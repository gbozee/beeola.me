---
title: Context Switching
date: "2019-02-22T16:05:56.281Z"
---

As someone who works both frontend and backend web technologies, I always find my self in situations where for example, I need to implement a functionality on the backend while working on the frontend. Often times this is very painful not because the task to be done is difficult but mostly because my train of thought is mostly impacted. 

Recently, I began an experiment. Because of my understanding of how the web api in question would be fetched, I have a particular module called `adapter.js`. This module fakes all the api calls that would be made throughout my application with JavaScript Promises.

An example could look like this.
Imagine I need an endpoint to call after collecting the login details of a form, and I am not ready to write the backend implementation, the implementation in the `adapter.js` would look like this

```js
    function loginUser(details){
        return new Promise((resolve, reject)=>{
            let sampleResponse = ...
            resolve(sampleResponse)
        })
    }
    ...
    export default {
        ...
        loginUser,
        ...
    }
```

Then my application entrypoint depends upon this `adapter` module. Or if I am feeling lazy, I just import this module wherever I need to use it.

I like this approach for many reasons
1. It delays/eliminate context-switching to the backend. Focus/Flow isn't easily lost.
2. The tasks I need to implement on the backend are obvious. This makes testing on the backend easy to think about because the expectation of what is required by the endpoint isn't ambiguous.
3. It doesn't matter what technology/library is used for data fetching, the success and the error case is isolated in the function, in my case `loginUser`
4. It is a good starting point for backend developers when transitioning to  frontend codebases since making api calls isn't foreign to them.

One obvious disadvantage with this approach is the possibility of making multiple similar calls, especially when [GraphQL]() is used. But this can be solved with a little work, depending on how paranoid one might be about performance.

Would love to hear more about anyone practicing this approach and what they think.