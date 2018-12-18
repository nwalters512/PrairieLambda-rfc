# PrairieLambda RFC
Working draft of a proposal to bring "serverless" lambdas to PrairieLearn

## Introduction

PrairieLearn is a LMS built in-house at Illinois. Compared to traditional LMSes, PrairieLearn allows far greater flexibility by allowing questions to be backed by arbitrary code. One can build questions that can generate and grade almost infinite variants of themselves, meaning that questions can be reused almost infinitely - across exams, semesters, and even courses.Traditionally, question code was written in JavaScript. With the introduction of the v3 question format (otherwise known as Freeform), question servers were written in Python to make available libraries that are widely used in engineering courses (NumPy, SymPy, matplotlib, etc.). This has generally resulted in a better question authoring experience for people building PrairieLearn content.

In addition to writing per-question code, v3 questions also allow questions to be built from reusable, composable elements. These elements, like questions, are backed by arbitrary Python code, and are able to render, parse, and grade themselves. Unlike in most LMSes, users aren't limited to writing questions of particular "types" (multiple choice, short answer, etc.). It's possible to combine multiple elements in one question, meaning that you can have a question that includes multiple choice options, a numerical input, and a file upload, for instance. The elements system has proven extremely powerful - people have built elements to render 3D models, input matrices, and render GraphViz DOT notation. PrairieLearn provides a comprehensive "standard library" of elements, but courses may also define their own for use in their questions.

Together, these systems have made possible a number of performance optimizations. Element renders can be treated as pure function calls, transforming question data into rendered HTML. Since these are idempotent, we can cache entire question panel renders and reuse them on repeat visits to a page. Additionally, we can pre-fork Python workers to virtually eliminate the cost of starting the Python interpreter.

## Limitations

So far, the primary limitation has been the runtime environment of question and element code. Specifically, they can only be written in Python, and they can only use libraries that have been pre-installed by PrairieLearn. So it isn't possible to use other languages, and it's also not possible to use any Python libraries that haven't been installed in the PrairieLearn.

The other main limitation is security. This is due primarily to our implementation: Python processes aren't isolated from each other. We could potentially fix this independently of the runtime restrictions, but we have the opportunity to kill two birds with one stone here.

## Prior Art

In PrairieLearn's external grading system, we solved a similar problem of securely executing arbitrary code. PrairieGrader jobs have to specify their environment to support a variety of languages; so far, we've had people grading Python, Java, C++, Haskell, Ocaml, Clojure, and more. To do so, they upload Docker containers and specify which contianers should be used to grade particular questions. To grade a question, a PrairieGrader worker downloads the submitted code, pulls the necessary Docker image, and starts a container with the submitted code. Once the container exits (or a timeout is exceeded), we pull grading results off a pre-defined path on the container's disk and return it to PrairieLearn for processing.

The growing "serverless" execution model tries to solve a similar problem. Developers specify "lambdas", which are essentially request handlers that are invoked to handle user requests. They can usually execute in somewhat-arbitrary environments with a variety of languages. We believe that this model will provide a good baseline to develop our own solution on top of.

## Requirements

Our solution must of course solve the problems we're trying to solve. That is, it must allow for arbitrary code execution in a variety of environments for a variety of languages. It must also be secure, isolating code from both the underlying operating system and from code for other questions or courses.

Any potential solution has to be *fast*. Question generation, rendering, and grading happens on almost every page load, and we can't start responding to a client request before we have fully rendered a question and its submissions. So we need to be able to do these operaitons quickly. Our existing caching system should be able to help with this.

Any solution has to work when running locally and has to allow for quick iteration for question development. The canonical way for course staff to work on course content is by running a local version of the PrairieLearn Docker image and mounting their course code into it. Our solution can't make this process significantly more complicated - that is, we can't do something like requiring users to deploy and host lambdas on a platform like AWS Lambda or Zeit Now. However, it would be acceptable to allow for us to rely on improved or more advanced tooling - for instance, by using Docker Compose to run multiple services locally.

Any solution should be able to scale as PrairieLearn does. That is, we shouldn't simply run all of these containers on the main PrairieLearn server. This appears to call for a distributed system of some sort.

## Interface

This section of the document outlines a hypothetical multi-language interface to course code.

We'll use HTTP as the foundation of this interface. It's a reliable way of doing IPC/RPC and has support for multiple concurrent requests, unline a solution based on `stdin`/`stdout`.

Course code will handle one or more of the following HTTP routes:

* `/question/{QID}`: invokes question code for the specified question
* `/element/{element name}`: invokes element code for the specified course element

The body of the request will contain the requested method to be invoked, as well as the the JSON-encoded `data` object that's already received by Python question/element code. For elements, it would also include the `element_html` string from Python. For example, here's a simplified request body for an element render:

```json
{
  "method": "render",
  "html": "<my-element attr=\"value\">hello, world!</my-element>",
  "data": {
    "params": {
      "number": 123
    }
  }
}
```

The response to a successful request should have a `200` response code, and the body should contain either a modified data object (for a `generate`, `prepare`, `parse`, `grade`, etc.) or rendered HTML (for a `render`).

If the specified question or element cannot be handled, a `404` code should be returned.

### Alternative interfaces

To avoid the overhead of establishing a new HTTP connection for each element/method, it would be possible to batch requests and responses. This could be done with a course server that only handles requests at the route `/`. A request would then contain the original data object and a JSON array of separate requests:

```json
{
  "data": {
    "params": {
      "number": 123
    }
  },
  "requests": [
    {
      "type": "question",
      "method": "grade",
    },
    {
      "type": "element",
      "method": "grade",
      "html": "<my-element attr=\"value\">hello, world!</my-element>",
   }
  ]
}

```
