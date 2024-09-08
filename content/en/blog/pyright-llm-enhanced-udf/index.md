---
title: "Suggesting Python Code Type Annotations with LLMs -- A Language Server Enhancement"
description: ""
lead: "In this blog, we introduce how we enhanced the Texera Python User-Defined Function（UDF) editor by integrating Large Language Models (LLMs) to suggest Python code type annotations, thereby overcoming the limitations of dynamic typing of traditional language servers like  Pyright and python-language-server (Pylsp)."
date: 2024-09-07T11:38:41-07:00
publishDate: 2024-08-30T00:00:00-07:00
lastmod: 2024-09-07T11:38:41-07:00
draft: false
weight: 50
contributors: ["Minchong Wu"]
affiliations: ["Computer Science, UC Irvine"]
---

## Motivation

The User-Defined Function (UDF) operators in Texera are crucial for allowing developers to customize their workflows. To improve the development experience, we need a python language server that can detect both syntax and semantic errors early in the coding process.

Previously, we relied on the Pylsp for basic language server functions like syntax checks and code completion. However, Pylsp had difficulty on detecting more complex semantic errors in user code, particularly with type mismatches. To enhance the user experience and provide more accurate error feedback on such semantic issues, we choose Pyright— a python language server with more powerful static type checking capabilities. Pyright helps developers write more robust and maintainable code by catching issues such as type mismatches early, significantly reducing debugging time.

<figure align="center">
  <a href="language_server_compare.png">
    <img src="language_server_compare.png" alt="Fig 1" style="width:70%">
  </a>
  <figcaption align = "center">
    <i>
      <b>Figure 1.</b> Comparing the Performance of the Two Language Servers on Basic Semantic Errors.
    </i>
  </figcaption>
</figure>

In Figure 1, there is a very basic comparison between two language servers on the same code. The yellow lines represent warnings, and the red lines represent errors. With Pylsp, it can recommend fixes with formatting suggestions from [PEP 8 style guide](https://peps.python.org/pep-0008/#style-guide), such as the [two blank line requirement](https://peps.python.org/pep-0008/#blank-lines) between functions. It is limited to formatting warnings, instead of code correctness. Also, as shown in the figure, when using Pylsp, there is only one red line, indicating that it detected only one meaningful semantic error. In contrast, Pyright correctly identified all the relevant errors, which is reflected by the additional red lines. For example, there is a type error in the second `def` block where `y = x + 5` and `x` is a string. This leads to a semantic error, as you cannot add a string and an integer. This demonstrates Pyright’s superior ability to detect semantic issues, making it a better choice for enhancing the UDF code editor.


## Challenge: Type Inference Without Type Annotations
The biggest challenge was that Pyright, while powerful, heavily relies on the presence of type annotations for accurate semantic error detection. Many Python code, especially the ones used in Texera Python UDFs, lacked the type annotations, leading to reduced accuracy of the language server in giving correct annotations. Therefore, the absence of accurate error detection could compromise code quality and user experience.

## Solution
**Type Annotation with LLMs**
To tackle the challenge of handling code without type annotations, we integrated Large Language Models (LLMs) into the UDF editor. This integration enables the LLM to automatically suggest type annotations, thereby enhancing the effectiveness of Pyright's static analysis. The LLMs generate type annotation suggestions based on the context of the code. Users need to decide whether to accept the suggestion, and if they do, the suggestion will be added after the argument.

We developed a backend RESTful API that interacts with OpenAI's GPT-4 API to generate these type annotation suggestions. Here are some key aspects of our implementation:

1. **API Design**: Our API integrates AI-assisted type annotations into the Texera UDF editor. The frontend Angular service, `AIAssistantService`, provides a method to request type annotations for given code snippets. It communicates with a backend Scala resource, `AIAssistantResource`, which handles authentication, processes requests, and interacts with the OpenAI GPT-4 API. The backend formats the code context into a specific prompt structure, sends it to OpenAI's chat completions API, and returns the suggested type annotations to the frontend. This design ensures secure, efficient communication between the user interface, our server, and the AI model, delivering intelligent type suggestions.
2. **Prompt Engineering**: We experimented with various prompts to optimize the LLM's performance. In the beginning, we simply asked it to provide type annotations for arguments, but we found that it often responded with explanations that we didn't need, which would cause errors if inserted into the code. So, we defined its task strictly to return in the form of `: type suggestion`, and we provided several different examples for the LLM to learn from to ensure it strictly follows our requirements, avoiding breaking the user's code.
3. **Prompt Effectiveness**: We found that prompts providing more context about the function's purpose and usage led to more accurate type suggestions. After testing, we found that most of the time, it can provide satisfactory results, but occasionally, unsatisfactory suggestions may occur. Therefore, users still need to have some judgment to decide whether to accept or reject the type suggestion.

<figure align="center">
  <a href="suggestion%20_UI.png">
    <img src="suggestion%20_UI.png" alt="Fig 2" style="width:70%">
  </a>
  <figcaption align = "center">
    <i>
      <b>Figure 2.</b> The type suggestion UI for either accept or decline.
    </i>
  </figcaption>
</figure>

This approach not only improve the accuracy of semantic error detection but also streamlines the development process by reducing the need for manual type annotations.

## Architecture overview
Figure 2 provides an overview of the architecture. The previously deployed components with Pylsp are shown in green, and the new structure shown in blue added in this blog. On top of the addition of the Pyright language server, two AI features have been introduced to enhance the performance of the Pyright language server.

<figure align="center">
  <a href="architecture_overview.png">
    <img src="architecture_overview.png" alt="Fig 3" style="width:70%">
  </a>
  <figcaption align = "center">
    <i>
      <b>Figure 3.</b> New architecture of connecting UDF editor with Language server.
    </i>
  </figcaption>
</figure>


## Supported features
For basic language server features, including hover, auto-completion, code linting, and go to definition, Pyright and Pylsp exhibit identical behavior. You can refer to the old blog for specific demonstrations: [Enhancing the UDF Editor by Adding Language Server Support](https://texera.github.io/blog/enhancing-the-udf-editor-by-adding-language-server-support/#challenges). Here, I will focus on how to use the two newly added AI features and whether these features have improved Pyright's ability to detect semantic errors.

<figure align="center">
  <a href="before_annotation.png">
    <img src="before_annotation.png" alt="Fig 4" style="width:70%">
  </a>
  <figcaption align = "center">
    <i>
      <b>Figure 4.</b> Semantic error detection of Pyright language server without type annotation.
    </i>
  </figcaption>
</figure>


As you can see, there are some semantic errors that Pyright is unable to detect in this image due to the lack of type annotations (In line 158, the create_user function should accept a string and an integer, but the user is calling this function with two strings. In line 173, the update_age function should accept an integer, but the user is calling this function with a string as an argument). We will compare Pyright's capability in detecting semantic errors on the same code after the type annotations have been added using our AI features.

1.**Add Type Annotation Button**: The feature gives a type suggestion of a single argument selected by the user. The users can interact with the UI and choose to accept or decline the suggestion. If the suggestion is accepted, it will be added as a type annotation in the correct location of the code; otherwise, the suggestion is ignored and the code and the code is not modified.

<figure>
  <a href="add_type_annotation.gif">
    <img src="add_type_annotation.gif" alt="Fig 5" style="width:100%">
  </a>
  <figcaption align = "center">
    <i>
      <b>Figure 5.</b> Examples of using "Add Type Annotation" button.
    </i>
  </figcaption>
</figure>


2.**Add All Type Annotations Button**: To accommodate the diverse needs of users, we offer a one-click feature designed to simplify the process. This feature gives type suggestions for all the arguments within the user-selected code. In this code snippet, each argument that requires a type annotation will be highlighted one by one, allowing the user to accept or decline each suggestion individually. By streamlining the process of identifying and annotating arguments, this feature significantly enhances convenience and reduces manual effort.

<figure>
  <a href="add_all_type_annotation.gif">
    <img src="add_all_type_annotation.gif" alt="Fig 6" style="width:100%">
  </a>
  <figcaption align = "center">
    <i>
      <b>Figure 6.</b> Examples of using "Add All Type Annotation" button.
    </i>
  </figcaption>
</figure>

## Empowering Pyright with Inferred Type Annotations
With all type annotations in place, users can fully enjoy Pyright's precise semantic type detection capabilities. As shown in the image below, compared to Figure 3, all semantic errors have been correctly identified by the Pyright language server.

<figure>
  <a href="after_annotation.png">
    <img src="after_annotation.png" alt="Fig 7" style="width:100%">
  </a>
  <figcaption align = "center">
    <i>
      <b>Figure 7.</b> Two more type errors are being detected after the type annotation being added.
    </i>
  </figcaption>
</figure>

## Limitation
As we all know, while AI is incredibly smart and convenient, it can still make mistakes in rare cases. Therefore, users cannot completely rely on the type suggestions returned by the LLM; they are merely suggestions. Users need to evaluate whether the suggestion is correct based on the design requirements of their code and make a careful choice to accept or decline it.

## Summary
In this blog, we showcased how we integrated the LLM with the Pyright language server to enhance the user experience of editing UDFs in Texera.

## Acknowledgements
Thanks to Prof. Chen Li, Yicong Huang, and the Texera team for their help in the project and in this blog.
