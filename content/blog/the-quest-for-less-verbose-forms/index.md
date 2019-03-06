---
title: The Quest for less-verbose Forms
date: "2019-03-06"
spoiler: Revealing flaws in React Codebases with Hooks
# draft: true
---

One of the least fun things to do in web development is handling form data. This problem is compounded further when you have to deal with a lot of forms within the application. While there are libraries in the [React](https://reactjs.org/) ecosystem that makes forms less painful e.g [Formik](https://jaredpalmer.com/formik/), [Redux Forms](https://redux-form.com/8.1.0/), I am not a fan of some of the abstractions provided by these libraries.

I decided to take a stab at writing a Form component that mirrored my vision of what a Form component library should look like.

Using [Storybook](https://storybook.js.org/) as a test bed for the ideas I wanted to achieve, I came up with the following stories

```jsx
...
// A sample schema representing the attributes of the form
let data = [
  {
    name: "email",
    type: "email",
    label: "Email",
    validate: value => value.includes("@")
  },
  { name: "first_name", label: "First name" },
  { name: "last_name", label: "Last name" },
  {
    name: "gender",
    options: ["Male", "Female"],
    type: "radio",
    label: "Gender"
  },
  {
    name: "age_range",
    options: [["", "Select Age"], ["2", "Two year old"], ["3", "3 Year hold"]]
  }
];
const Wrapper = ({ children }) => (
  <Box
    css={css`
      .simple-form {
        flex-direction: column;
      }
    `}
  >
    {children}
  </Box>
);

storiesOf("Forms", module)
  .add("Default Rendering", () => (
    <Wrapper>
      <Form
        fields={data}
        onSubmit={result => {
          console.log(result);
        }}
      />
    </Wrapper>
  ))
  .add("With Existing Values", () => (
    <Wrapper>
      <Form
        fields={data}
        data={{ email: "gbozee@example.com" }}
        onSubmit={result => {
          console.log(result);
        }}
      />
    </Wrapper>
  ))
  .add("Custom Layout", () => {
    return (
      <Wrapper>
        <Form
          fields={data}
          onSubmit={result => {
            console.log(result);
          }}
          render={(fields, button) => (
            <Flex flexDirection="column">
              <Flex>
                {fields.first_name}
                {fields.last_name}
              </Flex>
              <Box>{fields.email}</Box>
              <Flex>{fields.gender}</Flex>
              <Flex>{fields.age_range}</Flex>
              <Flex>{button}</Flex>
            </Flex>
          )}
        />
      </Wrapper>
    );
  })
  .add("Custom Elements", () => {
    return (
      <Wrapper>
        <Form
          fields={[...data, { name: "expectaton", component: TextArea }]}
          onSubmit={result => {
            console.log(result);
          }}
        />
      </Wrapper>
    );
  })
  .add("With Error Messages", () => {
    const errors = {
      first_name: "This field is required",
      email: {
        empty: "This field is required",
        invalid: "This email is invalid"
      }
    };
    return (
      <Wrapper>
        <Form
          fields={data}
          error_messages={errors}
          onSubmit={result => console.log(result)}
        />
        /
      </Wrapper>
    );
  })
  .add("Custom Components", () => {
    const formElements = {
      text: TextArea
    };
    return (
      <Wrapper>
        <FormProvider formElements={formElements}>
          <Form fields={data} onSubmit={result => console.log(result)} />
        </FormProvider>
      </Wrapper>
    );
  });

```

Breaking down the above snippet,

```js
let data = [
  {
    name: "email",
    type: "email",
    label: "Email",
    validate: value => value.includes("@")
  },
  { name: "first_name", label: "First name" },
  { name: "last_name", label: "Last name" },
  {
    name: "gender",
    options: ["Male", "Female",'Any'],
    type: "radio",
    label: "Gender"
  },
  {
    name: "age_range",
    options: [["", "Select Age"], ["2", "Two year old"], ["3", "3 Year hold"]]
  }
];
...

const errors = {
    first_name: "This field is required",
    email: {
        empty: "This field is required",
        invalid: "This email is invalid"
    }
};
```

Usage of the Form component follows a simple convention and assumes certain defaults. From the data above, the following premise was implied.

1. if a field's `type` property doesn't consist of any of the supported types `text,email,radio,checkbox,radio,select`, a default `text` input field is implied.
2. If a field's `type` property doesn't exist but the `options` property does, then a `select` field is implied
3. For a `select` field, the `options` property could match any of the following format

   a. `['First', 'Second', 'Third']`

   b. `[['First','1st'],['Second','2nd'],['Third','3rd']]`

   c. `[{label:"1st", value:'First'}, {label:"2nd", value:'Second"}]`

4. The `name` property used in the `data` array must exist as a property in the `errors` object if an error message is to be displayed for such field.

5. If a field has an error message, the default condition is that the field shouldn't be empty. For any custom error message to be applied, the corresponding record in the `data` array must have a `validate` property which consist of the actual validation logic for that field.

_it should be noted that none of the assumptions provided above are set in stone and based on new requirements down the line, they would most likely be revised._

The first story captures a simple test case for how the form should work.

```jsx
<Form
  fields={data}
  onSubmit={result => {
    console.log(result);
  }}
/>
```

In this case, no default/initial data of the form exists and also no validation is implied. On submitting this form, the result of the fields is available from the `onSubmit` props

The second story provides initialValues for some of the form fields.

```jsx
<Form
  fields={data}
  data={{ email: "gbose@example.com" }}
  onSubmit={result => {
    console.log(result);
  }}
/>
```

In this case, only the fields provided by the `data` props are used as initial values, The remaining are assumed to be empty

The fifth story accounts for when an error should be displayed on the form.

```jsx
 const errors = {
      first_name: "This field is required",
      email: {
        empty: "This field is required",
        invalid: "This email is invalid"
      }
    };
    ...
    <Form
        fields={data}
        error_messages={errors}
        onSubmit={result => console.log(result)}
    />
```

an `error_messages` props is provided to capture the errors that should be displayed.

The third story provides an escape hatch for controlling how the form fields should be positioned.

```jsx
<Form
  fields={data}
  onSubmit={result => {
    console.log(result);
  }}
  render={(fields, button) => (
    <Flex flexDirection="column">
      <Flex>
        {fields.first_name}
        {fields.last_name}
      </Flex>
      <Box>{fields.email}</Box>
      <Flex>{fields.gender}</Flex>
      <Flex>{fields.age_range}</Flex>
      <Flex>{button}</Flex>
    </Flex>
  )}
/>
```

The fourth story gives room for the ability to use a custom component that isn't supported by default.

```jsx
const TextArea = props => <textarea {...props} />;
...
 <Form
    fields={[...data, { name: "expectaton", component: TextArea }]}
    onSubmit={result => {
    console.log(result);
    }}
/>
```

as long as these custom components have an `onChange` props, it should work and render without any issues. Any addition props required by this component are provided as part of the field's data

The last story considered, which is actually the driving factor for building this component, is the ability to pre-register all the supported components the application would use.

```jsx
 const formElements = {
      text: TextArea
    };
    ...
<FormProvider formElements={formElements}>
    <Form fields={data} onSubmit={result => console.log(result)} />
</FormProvider>
```

This could include form field elements not natively supported. So for example, If an autocomplete Input field is implemented and needs to be globally exposed as the type `autocomplete-input`. It just needs to be registered up-front and then a sample `data` array could look like this

```js
const data = [
  {
    name: "address",
    type: "autocomplete-input",
    endpoint: "https://google-api..."
  }
];
```

The goal is to keep this Component as lean as possible while more use-cases are being figured out.

A first stab at the implementation looks like this.

```jsx
import React, { cloneElement, useState, useReducer } from "react";

const LabelWrapper = ({ children, label, name, errorMessage }) => (
  <div>
    {label && <label htmlFor={name}>{label}</label>}
    {cloneElement(children, { id: name })}
    {errorMessage && <div style={{ color: "red" }}>{errorMessage}</div>}
  </div>
);

//Custom Input component.
const Input = ({ onChange = () => {}, options, ...props }) => (
  <LabelWrapper {...props}>
    {Array.isArray(options) ? (
      <>
        {options.map(option => (
          <LabelWrapper label={option} name={option}>
            <input
              {...props}
              id={option}
              onChange={e => onChange(e.target.value, e)}
            />
          </LabelWrapper>
        ))}
      </>
    ) : (
      <input {...props} onChange={e => onChange(e.target.value, e)} />
    )}
  </LabelWrapper>
);
//Custom Select Component
const Select = ({ onChange = () => {}, label, options = [], ...props }) => (
  <LabelWrapper {...props}>
    <select {...props} onChange={e => onChange(e.target.value, e)}>
      {options.map((option, index) => {
        let data = option;
        if (typeof data === "string") {
          data = { label: option, value: option };
        }
        if (Array.isArray(option)) {
          data = { value: option[0], label: option[1] };
        }
        return <option value={data.value}>{data.label}</option>;
      })}
    </select>
  </LabelWrapper>
);
//Button component used by the form
const Button = ({ text = "Submit", ...props }) => (
  <button {...props}>{text}</button>
);
// An object of all the default components supported and registered in the context.
let defaultFormComponents = {
  text: Input,
  email: Input,
  radio: Input,
  select: Select,
  checkbox: Input
};
const FormContext = React.createContext(defaultFormComponents);
//The Provider component for overriding the default Form fields.
export const FormProvider = ({ formElements = {}, children }) => {
  let components = { ...defaultFormComponents, ...formElements };
  return (
    <FormContext.Provider value={components}>{children}</FormContext.Provider>
  );
};

// The reducer logic used in updating the state of the forms.
function reducer(state, action) {
  if (action.type == "change") {
    return { ...state, [action.field]: action.value };
  }
  throw new Error();
}

//The actual form component
export const Form = ({
  fields = [], // the form fields array
  render, //An optional callback to manually change the form placement
  error_messages = {}, // error_messages to be displayed for the form fields
  onSubmit = () => {}, // callback with the response after validation occurs
  data, // initial values of the form fields
  ButtonComponent = Button, // Overwriting the button component used in submitting the form (might be moved to the form provider)
  formProps = { noValidate: true, className: "simple-form" } // additional props to be applied to the form
}) => {
  //Depending on if initial values for the form fields are provided, we need to set all the form fields value to an empty string
  let formValues = {};
  let errorMessages = {};
  fields.forEach(field => {
    formValues[field.name] = "";
  });
  if (Boolean(data) && typeof data === "object") {
    formValues = { ...formValues, ...data };
  }
  let [state, dispatch] = useReducer(reducer, formValues); // initializing the state with the default values for all fields.
  let [displayError, toggleError] = useState(false); // this state determines when an error message is to be displayed. For now, it is triggered only when the submit button is clicked.

  //We need to generate an object which consist of the validation logic for each field depending on if error messages are provided per field or not. This is a naive implementation and would definitely be refactored to support a more fine-grained use-case
  Object.keys(error_messages).forEach(field => {
    let error = error_messages[field];
    if (typeof error === "string") {
      return value => (!Boolean(value) || value.length === 0) && error;
    }
    let func = value => {
      let empty = (!Boolean(value) || value.length === 0) && error.empty;
      let validateFunc = fields.find(x => x.name === field).validate;
      let invalid = undefined;
      if (validateFunc) {
        invalid = !validateFunc(value) && error.invalid;
      } else {
        throw new Error(
          `${field} field is missing the "validate" function property`
        );
      }
      return empty || invalid;
    };
    errorMessages[field] = func;
  });

  //We read the supported field components from the context and generate a new array consisting of the name of the field as well as the component matching that field.
  let formComponentsContext = React.useContext(FormContext);
  let formFields = fields
    .map(field => {
      let result = {
        ...field
      };
      //this is where we handle when the `type` property isn't set.
      if (!Boolean(result.type)) {
        if (Array.isArray(result.options)) {
          result.type = "select";
        } else {
          result.type = "text";
        }
      }
      return result;
    })
    .map(field => {
      let Component = formComponentsContext[field.type];
      let { type, component, ...rest } = field;
      // If a custom component is provided, we replace the generated component with the passed component.
      if (component) {
        Component = component;
      }
      // The error message function for each field is pulled and would be appended as a props for each of the field's component. This means all the components would have an `errorMessage` prop for displaying the error message. If no error message was provided, it returns undefined. The field component is then responsible for handling when the `errorMessage` prop is `undefined`

      let errorMessageFunc =
        errorMessages[field.name] ||
        function() {
          return undefined;
        };
      //If the Component to be rendered doesn't exist or is null, instead of throwing an error, `undefined` is rendered instead.
      return {
        name: field.name,
        component: Component ? (
          <Component
            key={field.name}
            {...rest}
            value={state[field.name]}
            type={type}
            onChange={
              value => dispatch({ type: "change", value, field: field.name }) // right now, only the onChange event is being handled. others could definitely be supported.
            }
            errorMessage={displayError && errorMessageFunc(state[field.name])} // only display the error message when the `displayerror` state is true and if the `errorMessage` function isn't undefined.
          />
        ) : (
          undefined
        )
      };
    });

  // The form submission logic to be triggered when the form submission attempt is made.
  const onFormSubmit = e => {
    e.preventDefault();
    // We are checking to ensure no error message was triggered by any of the fields that require validation.
    let hasErrors = Object.keys(errorMessages).filter(field =>
      Boolean(errorMessages[field](state[field]))
    );
    if (hasErrors.length === 0) {
      //Since this is the first draft, I'm not sure of any additional parameters to be sent down.
      onSubmit(state, { toggleError });
    } else {
      //An error was observed so we need to give a visual cue to the user.
      toggleError(true);
    }
  };

  const submitButton = <ButtonComponent type="submit" />;
  //In this case, we are using the default rendering behaviour for the form,
  if (!Boolean(render)) {
    return (
      <form {...formProps} onSubmit={onFormSubmit}>
        {formFields.map(x => x.component)}
        {submitButton}
      </form>
    );
  }
  let formRenderFields = {};
  formFields.forEach(field => {
    formRenderFields[field.name] = field.component;
  });
  //In order to give the user partial control of the rendering process, an object representing the fields and their corresponding component is sent as parameters to the `render` function.
  return (
    <form {...formProps} onSubmit={onFormSubmit}>
      {render(formRenderFields, submitButton)}
    </form>
  );
};
```

**While this implementation needs some refactoring, It appears to solve to a large extent, most of the use-cases I frequently run into and pushes most of the work to the individual Form Elements that make up the form**

Some use cases to consider in my implementation is leveraging a validation-schema library like [Yup]() instead of having to depend on the `validate` property on each item in the `fields` props as well as the `errors` props passed. 
