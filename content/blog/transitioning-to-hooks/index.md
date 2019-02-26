---
title: Transitioning to React Hooks
date: "2019-02-22T16:05:56.281Z"
draft: true
---
```js
class Checkout extends React.Component{
    ...
    handleDurationChange = duration => {
        if (typeof duration === "string") {
            this.setState({ duration }, () => {
                if (this.state.display === "payment") {
                    this.generateNewPayment();
                }
            });
        }
    }
    generateNewPayment = (successCallback = () => {}) => {
        let { coupon = "" } = this.state;
        this.props
        .updateCoupon({ ...this.state, plan: this.state.currentPlan })
        .then(data => {
            successCallback(true, data.discount === 0);
            this.setState({
                discount: data.discount,
                isInvalidCoupon: coupon ? data.discount === 0 : false,
                isCouponApplied: data.discount > 0
            });
        });
    };
  ...
}
```

```js

const Checkout = ({plans,updateCoupon})=>{
    let [currency, setCurrency] = useState(...)
    let [duration, setDuration] = useState(...)
    ...
    const handleDurationChange = value => {
        if (typeof value === "string") {
            setDuration(value);
            if (display === "payment") {
                generateNewPayment({ duration: value });
            }
        }
    };
    const generateNewPayment = ( existingData = {}, successCallback = () => {} ) => {
        updateCoupon({
            currency,
            duration,
            coupon,
            plan: currentPlan,
            ...existingData
        }).then(data => {
            successCallback(true, data.discount === 0);
            setDiscount(data.discount);
            setIsInvalidCoupon(coupon ? data.discount === 0 : false);
            setIsCouponApplied(data.discount > 0);
        });
    };
}
```

One benefit observed when transitioning to hooks is that text editors like Visual Studio Code highlight functions that aren't in use. In a class component, such functions are attached as methods and even when they aren't in use, You as the developer do not know and so they remain there as `zombies`