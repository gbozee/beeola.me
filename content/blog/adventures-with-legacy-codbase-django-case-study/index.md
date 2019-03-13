---
title: Adventures with Legacy Code (A Django case study)
date: "2019-03-13"
spoiler: A case study on the process involved in adding a new functionality to a legacy Django codebase.
keywords: python, django, database, legacy, many-to-many-relationship, tdd, testing
---
Legacy codebases are everywhere. As software developers, we need to be comfortable with it. We need to understand it in order to improve on it. 

I was tasked with providing improvements to a specific part of the codebase that was implemented over 4 years ago. This blog post goes into full details to explain the problem and how the problem was solved. 

## The Backstory
There is a `Booking` model that has a foreign key relationship with both a `Customer` model and a `Trainer` model. Both `Customer` and `Trainer` shared the same base Django `User` model

```py

class BookingSession(BookingSessionMixin, TimeStampedModel):
    ...
    start = models.DateTimeField()
    price = models.DecimalField(max_digits=10, decimal_places=2)
    no_of_hours = models.DecimalField(max_digits=5, decimal_places=2)
    booking = models.ForeignKey("bookings.Booking", null=True)
    status = models.IntegerField(default=NOT_STARTED, choices=SESSION_STATUS)
    objects = BookingSessionManager()
    ...

class Booking(BookingDetailMixin, BookingMixin, TimeStampedModel):
    order = models.CharField(max_length=12, primary_key=True, db_index=True)
    customer = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.SET_NULL,
        related_name="orders",
        null=True,
        blank=True)
    status = models.IntegerField(
        choices=BookingMixin.BOOKING_STATUS, default=BookingMixin.INITIALIZED)
    trainer = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.SET_NULL,
        related_name="t_bookings",
        null=True,
        blank=True,
    )
    objects = BookingManager()
    ...

class WalletTransaction(TimeStampedModel):
    ...
    type = models.IntegerField(choices=WalletTransactionType.CHOICES)
    amount = models.DecimalField(
        default=Decimal(0), max_digits=16, decimal_places=8)
    transaction_type = models.CharField(
        choices=WalletTransactionType.TYPES,
        default=WalletTransactionType.INCOME,
        max_length=15,
    )
    amount_paid = models.DecimalField(
        max_digits=16, decimal_places=8, default=Decimal("0.00"))

    booking = models.ForeignKey(
        "bookings.Booking",
        related_name="wallet_transactions",
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
    )
    objects = WalletTransactionManager()

```
_Most of the fields have been removed in order to prevent noise. I also omitted the `User` model_

The `Booking` model kept track of all the bookings/lessons provided, the `BookingSession` model kept track of the individual sessions assigned to a `booking` instance and the `WalletTransaction` model kept track of the transactions related to each booking.

This codebase is currently in production. Thousands of records have been created using this approach and a workflow for less technical individuals have been built on top of this approach.

## The problem
The above model implementations was built to support the scenario where a `trainer` could only have one `client/student` connected to a `booking` instance at any point in time. It doesn't prevent the possibility of the trainer having multiple clients but multiple booking instances would need to be created to keep track of these other clients.

**In easier terms, a trainer can only work with a single client at any period in time.**

But in reality, this isn't always the case. A trainer could work with a group of different clients together within a specific time.

Trying to accomplish the use-case for multiple clients isn't impossible, but results in a **space  problem (Duplicate records)**. With the current implementation, booking instances would need to be created for each client with association to the tutor. This approach breaks down when attempting to create the sessions to be linked to the booking instance. Based on the above model implementation, it is only possible to have a single `booking` instance attached to a session record. 

While some of this hurdles could be resolved, I was interested in a solution that achieved the following.

1. Avoid having to create a new table/model
2. From the trainer's point of view, a lesson with multiple client at the same time is no different from a lesson with a single client within a specific time frame.  So if a trainer had two previous bookings with individual clients and a `group` booking with a number of other clients, calling `trainer.t_bookings.count()` should return `3` results.
3. From a client's point of view, the client expects a `booking` instance that links the specific client to the trainer irrespective of the other clients withing the same lesson attended. So if a client had two previous individual lessons and a group lesson with other clients, calling `client.orders.count()` should result in `3` as the result.
4. The solution shouldn't break any existing implementation since like I mentioned earlier, the codebase is already in production.
5. Since trainers are paid after every booking, in the case of a group lesson/booking, they should be paid the lump sum amount. For example, if a client pays $35 to attend a group lesson and 7 clients attended the class, since the trainer earns 70% of the cost of each booking, then the trainer should expect to be paid `(35*7) * 0.7 = $171.5` at the end of the group lesson.


Been able to layout what the solution should look like, makes it easy to come up with test cases.

```py
class NewBookingModelTestCase(TestCase):
    ...
    def create_initial_booking_with_tutor_and_no_clients(self):
        """
        self.ts1 represents the tutor instance. There are two sessions attached to this
        booking and the tutor is to earn 15% of the total sum of what all the clients pays
        for the booking
        """
        booking = b_models.Booking.create_group_booking(
            self.ts1,
            "October Lessons",
            sessions=[{
                'start': datetime(2018, 10, 1),
                'price': 2000,
                "no_of_hours": 1
            }, {
                'start': datetime(2018, 10, 3, 12, 30),
                'price': 2000,
                'no_of_hours': 1
            }],
            split=15)

    def test_creating_a_group_lesson_booking(self, mock_mail):
        # helper method to create multiple clients
        self.client, new_client, third_client = self.create_clients()

        # helper method to create a booking with a tutor, consisting of 2 sessions.
        booking = self.create_initial_booking_with_tutor_and_no_clients()
        # ensuring that there are two sessions attached to the booking on creation.
        self.assertEqual(booking.bookingsessions.count(), 2)
        
        # the booking instance consist of a `add_to_lesson` method to add individual clients.
        booking.add_to_lesson(self.client)
        booking.add_to_lesson(new_client)
        # a second parameter which signifies the number of sessions allowed for a particular 
        # client. if nothing is passed, it is assumed that the client is allowed for all the 
        # sessions. In this case, the third client is only allowed to take a single class instead
        # of the two classes provided by the booking
        booking.add_to_lesson(third_client, 1)

        # providing a way to know the number of group lessons a particular trainer has
        self.assertEqual(self.trainer.group_lessons.count(), 1)

        # after fetching a specific booking instance, provide a method to get all the clients that 
        # attended the booking
        for user in [self.client, new_client, third_client]:
            self.assertIn(user, booking.get_users)

        # since each session costs 2000 and only 2 of the 3 clients are allowed to 
        # attend both sessions supported, the 3rd client would only attend one of the two sessions, 
        # the total cost of the booking from the point of view of the tutor is (2000 * 2 ) * 2 + (2000) = 10000
        self.assertEqual(booking.total_price, 10000)

        # Each client has a way to view transactions associated with them. Since the clients aren't 
        # related, they each each have a single transaction record.
        self.assertEqual(self.client.wallet.transactions.count(), 1)
        self.assertEqual(new_client.wallet.transactions.count(), 1)

        # for each transaction associated with a client, the amount spent can be accessed.
        # So in the case of the third client who only booked for a single session, 2000 was spent.
        self.assertEqual(
            third_client.wallet.transactions.used_to_hire().first().total,
            2000)
        ...
        # Now imagine the scenario where the third client decides to pay for the last session provided by the booking after the booking has started, 
        third_booking_instance = booking.get_booking(third_client)
        third_booking_instance.update_booking()
        ...
        # The total amount of the booking becomes 12000 and the amount spent by 
        # third client for the booking becomes 4000
        self.assertEqual(booking.total_price, 12000)
        self.assertEqual(
            third_client.wallet.transactions.used_to_hire().first().total,
            4000)

        # Trainers can only get paid after the booking has been closed
        # close booking
        booking.close_booking()
        ...
        self.assertEqual(
            b_models.Booking.objects.get(order=booking.order).status,
            b_models.Booking.COMPLETED)
        ...
        self.assertEqual(booking.bookingsessions.completed().count(), 2)
        # Since the trainer is to earn 15% of the total booking amount, that results to
        # 1800
        self.assertEqual(
            booking.wallet_transactions.tutor_earning().first().total, 1800)
        ...
```

*Some assertions were intentionally omitted so as to make the test scenario easy to follow.*

This is one of five different test scenarios that was accounted for in order to ensure the new functionality works as expected.

From the above test case, One begins to see new functionalities emerge that weren't possible before.
Some of these functionalities includes

1. The ability to merge multiple existing bookings into a single booking.
2. The ability get clients to pay for just some of the sessions provided by the booking but not all. This could be thought of as a test-trial for clients who weren't initially convinced, but were worn over after the trial. 

*Test cases were provided for the scenarios provided above but were intentionally avoided in this blog post.*  


## The solution
I needed to provide a way whereby a `booking` instance could support multiple `sub-bookings` within it. The individual `sub bookings` have no `trainer` attached to them. To get the `trainer` for a particular sub-booking, you need to call the parent booking to get the trainer.

Also, the parent booking houses all the sessions. Since a `sub-booking` represents a client's `booking` instance, trying to access the `sessions` available is a little tricky since not all clients choose to attend all the sessions provided by the trainer.

Speaking with code, we can solve the problem of supporting `sub_bookings` by adding the following additional fields to the `Booking` model

```py
class Booking(BookingDetailMixin, BookingMixin, TimeStampedModel):
    ...
    bookings = models.ManyToManyField('self', symmetrical=False)
    is_group = models.BooleanField(default=False)
    ...

```
The `self` parameter tells Django that there is a many to many relationship within the same model instance. As for the `symmetrical=False` parameter, more information about it can be found [here](https://docs.djangoproject.com/en/2.1/ref/models/fields/#django.db.models.ManyToManyField.symmetrical) 

The `BookingSession` model also needs to be updated with the following field

```py
class BookingSession(BookingSessionMixin, TimeStampedModel):
   ...
   group_bookings = models.ManyToManyField('bookings.Booking', related_name="sessions")
   ...
```
This field is how `sessions` for each `sub-booking` is being tracked. 

The code implementation to the test case is provided below with comments explaining how it works

```py
class Booking(BookingDetailMixin, BookingMixin, TimeStampedModel):
    ...

    def add_to_lesson(self, client, sessions_to_book=None, expired=False):
        """Add client to group lesson"""
        # A sub booking is created without a tutor attached to it by passing `set_tutor` = True
        sub_booking = Booking.create(
            user=client, ts=self.ts, is_group=True, set_tutor=True)
        
        # A list of all the primary keys of the sessions attached to the parent booking is fetched
        sessions = self.bookingsession_set.values_list('pk', flat=True)
        
        # sessions_to_book tracks the number of sessions available to a sub_booking. If none is passed
        # it is assumed that all sessions are available to the sub_booking
        if sessions_to_book:
            sessions = sessions[:sessions_to_book]
        
        # Remember the `group_bookings` many_to_many field that was added to the `BookingSession` model, 
        # We are interested in the intermediate table created by Django that houses the primary key
        # between the Booking model and the BookingSession model.

        # so instead of creating the booking sessions directly and duplicating records, we end up 
        # creating new instances of the `intermediate table` with the existing `session_id` as well as the `booking_id`. 
        # This is the tricky part and what makes everything fit together. More info provided below.
        the_model = BookingSession.group_bookings.through
        result = the_model.objects.bulk_create([
            the_model(bookingsession_id=x, booking=sub_booking)
            for x in sessions
        ])
        ...
        # the subbooking is then added to the main booking
        self.bookings.add(sub_booking)

    @classmethod
    def create(cls, ts=None, user=None, tutor_id=None, **kwargs):
        order = generate_code(cls) 
        set_tutor = kwargs.pop('set_tutor', False)
        obj, _ = cls.objects.get_or_create(
            order=order, ts=ts, user=user,
            tutor_id=None if set_tutor else (tutor_id or ts.tutor_id),
            **kwargs,
        )
        return obj

    @classmethod
    def create_group_booking(cls, tutor_skill, description, sessions=[], split=15):
        # this is the method called when creating a group booking to house multiple sub bookings
        booking = cls.create( ts=tutor_skill, booking_level=split, remark=description,
            wallet_amount=0, tutor=tutor_skill.tutor)
        BookingSession.objects.bulk_create(
            [BookingSession(**x, booking=booking) for x in sessions])
        booking.compute_first_session()
        return booking
```

The tricky part that helps us with avoiding duplicate sessions for the sub_bookings is this
```py
    ...
    the_model = BookingSession.group_bookings.through
    result = the_model.objects.bulk_create([
        the_model(bookingsession_id=x, booking=sub_booking)
        for x in sessions
    ])
    ...
```
To understand this, we need to understand how many-to-many relationship tables are created.
In any many-to-many relationship setup, 3 tables are created. In the above example case, the `booking` table, the `booking_session` table and finally, the `booking_booking_session` table.

The `booking_booking_session` table is what is referenced when using the following django method `BookingSession.group_bookings.through`

it keeps track of the `primary_key` for both the booking as well as the booking_session. Bear in mind that a `primary key` only exists for a record if that record exists. It is impossible to have a `primary_key` for a record that doesn't exists. This is how relational databases operate.

In our use-case, When the `base_booking` is created with the `sessions` attached to it, we can assume that the `booking_booking_session` table consist of the following record


| pk  | booking_id  | booking_session_id  |
| --- |:-----------:| -------------------:|
| 1   | 1           | 1 |
| 2   | 1           | 2 |

Now, when creating a `sub-booking`, we do not want to create  new `sessions` because they already exist. We need to create entries in this `booking_booking_session` table so that we can assign the `primary_key` of the `sub_booking` to the `primary_key` for each of the sessions to be connected to.
So if the `primary_key` of a `sub_booking` is `2` and we need to link to all the sessions provided by the `parent_booking`, the `booking_booking_session` table would look like this

| pk  | booking_id  | booking_session_id  |
| --- |:-----------:| -------------------:|
| 1   | 1           | 1 |
| 2   | 1           | 2 |
| 3   | 2           | 1 |
| 4   | 2           | 2 |

What this means then is if we need to get all the `sessions` for a particular `sub_booking`, the following query would suffice

```py

sub_session_sessions = [x.booking_session for x in BookingSession.group_bookings.through.objects.filter(booking_id=sub_booking.pk)]

```

**NB: `BookingSession.group_bookings.through` points to the `booking_booking_session` table.**

The remaining relevant parts of the model to get the test case to pass is shown below

```py
class Booking(BookingDetailMixin, BookingMixin, TimeStampedModel):
    ...
    @property
    def total_price(self):
        """Price of booking based on summation of all the booking sessions"""
        
        # This function is inefficient since it results in multiple hits to the db.
        # but isn't a concern at the moment
        if self.bookings.exists(): 
            return self.total_group_price()

        # existing functionality provided by existing codebase.
        if hasattr(self, "session_total"):
            return self.session_total
        return self.all_aggregates.get("price__sum")

    def total_group_price(self):
        # since each sub_bookings are bookings, they all have access to the `total_price` property
        return sum([x.total_price for x in self.bookings.all()])

    @property
    def get_users(self):
        # a helper method to get the list of clients connected to a parent_booking.
        result = [x.user for x in self.bookings.all()]
        if result:
            return result
        return [self.user] if self.user else []

    # this method is supposed to only be called by sub_booking instances
    def update_booking(self):
        # for a sub_booking to access its parent booking, it needs to call the `reverse` relationship
        # provided by django for ManyToManyFields fields.
        base_booking = self.booking_set.first()

        # getting all the sessions attached to the parent_booking
        all_sessions = base_booking.bookingsession_set.values_list('pk', flat=True)
        # getting all the sessions currently available to the sub_boking
        local_sessions = self.bookingsessions.values_list('pk', flat=True)
        # using set theory to figure out the difference i.e remaining session not availabe in the local session.
        set_value = list(set(all_sessions).difference(set(local_sessions)))
        # adding the remaining sessions to the sub_booking allowed sessions
        the_model = BookingSession.group_bookings.through
        the_model.objects.bulk_create(
            [the_model(bookingsession_id=x, booking=self) for x in set_value])
        ...
    ...

```

## Summary
Understanding the problem was the most important aspect of the experience. This resulted in coming up with useful test-cases that mirrored the solution to what the problem solved. 

Testing was especially important in this case because, it was what drove the implementation as well as a acting as a check, ensuring that legacy functionalities weren't broken as a result of the new functionalities added.

Understanding the tools/framework, provided clarity in coming up with solutions to the problem at hand.
