---
layout: post
title: "Generative UI: Chatbot Inline Citations and Components"
date: 2025-11-13
---

<style>
.citation {
  border-bottom: 1px dotted #0969da;
  cursor: help;
}
.citation:hover {
  background-color: #fff3cd;
}
</style>


Most generative AI applications today use a chatbot interface to interact with users. Natural language is a familiar input for users and LLMs are specifically designed to handle it well. Today's LLMs can do much more than just generate natural language though, and we would do well to leverage their full capabilities to build richer user experiences.

For example, chatbots greatly benefit from the ability to cite sources for the information they provide. As we discussed in the previous blog, [Anthropic's Citation API](https://claude.com/blog/introducing-citations-api) shows that a model is capable of mixing natural language generation with xml-like citation tags to indicate the source of specific claims. You don't see these xml-like tags in the API output, because Anthropic post-processes the output to remove the tags.

I find the idea of mixing natural language content with xml-like tags quite inspiring and have been experimenting with it myself. In this blog, I'll outline a simple but powerful architecture for building chatbots that can generate inline citations and other rich components such as links, images, and call-to-actions.

## The system prompt

A good place to start understanding any agentic architecture is the system prompt.
I always strive to keep the system prompt as simple as possible. The architecture I will outline here leads to a particularly simple system prompt. Here's the one I use for this architecture:

>You are Dan, an AI assistant who is an expert at suggesting travel tips.
>Your purpose is to show off the ability to embed custom html components in your answers.
>You can include the following html components in your answers to enhance the user experience:
>
>1. Maps: Use this component to suggest locations to visit.
>
>- Inputs:
>
>    - location - The name of the location to visit.
>
>- Example usage:
>```
>I suggest you visit the Eiffel Tower in Paris. <Maps location="Eiffel Tower, Paris" />
>```
>
>- The user will see an embedded Google Maps iframe of the location you suggested.
>
>Always use the Maps component when suggesting locations.

The agent is very simple. It has no tools, so you can barely call it an agent. Yet, it can display any location in the world to its users, just by outputting an xml-like tag: `<Maps location="..." />`.

Of course, we need to do more work to actually render these tags in the user interface. This is the responsibility of the developer; not the LLM. The LLM has a simple job: generate natural language text and sprinkle in some xml-like tags when appropriate.

## The magic happens in the frontend

So, how do we render these xml-like tags in the user interface? The answer is simple: we parse the LLM output and replace the tags with actual components.
Now I'm no fan of building custom parsers. Luckily, we can rely on the community to help us out:

- react-markdown: A popular React component for rendering markdown content. It has a plugin system that allows us to extend its functionality.
- rehype-raw: A rehype plugin that transforms xml-like tags into React components.
- rehype-sanitize: A rehype plugin to whitelist only the tags we want to allow, preventing malicious code injection.

The relevant code snippet is:
```jsx
<ReactMarkdown
    remarkPlugins={[remarkGfm]}
    rehypePlugins={[
    rehypeRaw,
    [rehypeSanitize, sanitizeSchema],
    ]}
    components={components}
>
    {children}
</ReactMarkdown>
```
where
```jsx
const components = {
  maps: ({ location }) => <Maps location={location} />,
};
const sanitizeSchema = {
    tagNames: ['maps'],
    attributes: {
        maps: ['location'],
    },
};
```

Having set this up, all that remains is to implement the `Maps` component. Here we can make use of an iframe to embed Google Maps:

```jsx
const Maps = ({ location }) => {
    const mapsUrl = `https://www.google.com/maps?q=${encodeURIComponent(location)}&output=embed`;

    return <iframe src={mapsUrl} />;
};
```

That's it! Feel free to spice it up with some styling for flair.

Now if the chatbot replies with:
>You should visit <Maps location="Amsterdam, Netherlands" /> and get a stroopwafel.

We see
![Amsterdam]({{ site.baseurl }}/assets/images/amsterdam.png)

Fantastic!

## Powerful in its simplicity

I'm quite fond of this architecture. We needed no tools, no complex system instructions, and yet the model can now interleave any react component at any place inside its natural language output.

Because this base principle is so simple, we can combine it with other capabilities. Even if your system prompt is complex due to business logic, I'm pretty hopeful that asking the model to sprinkle some xml-like tags in its output will not hamper its performance.

## On to citations using Text Fragments

The Maps example was fun, but let's move on to something more useful: citations.
We apply the same principle as before, adding a `Cite` component to our system prompt. Below the `Maps` component instructions, we add:

>2. Cite: If the user uploaded a document, use `Cite` to reference a snippet of text within the document.
>
>- Inputs:
>
>    - documentKey - Each document is preceded with `Document key: <documentKey>`. Use this to identify the document you are citing.
>    - page - The page number where the cited text can be found. Zero-indexed.
>    - starttext - The beginning of the cited text. Only include the first 3 words.
>    - endtext - The end of the cited text. Only include the last 3 words.
>
>- Example usage:
>```
>Maldecena is known for his work on the AdS/CFT correspondence. <Cite documentKey="arXiv:hep-th/9711200" page="3" startText="In the last" endText="string or M-theory."/>
>```
>
>- The user will see a clickable badge with the filename. Clicking on it will open a sheet displaying the document at the correct page and highlighting the cited text.
>
>Always use the Cite component when citing documents.

This Cite component has a few more inputs. We chose these cleverly to achieve the following goals:
- A short documentKey allows the LLM to express which document it is citing without using too many tokens. We need to make sure to pass this documentKey together with the document when calling the LLM.
- The LLM does not need to cite full verbatim text, which we cannot trust and is needlessly slow and expensive. Rather, we just ask for the beginning and end of the cited text.

Here's pseudo code for the Cite component:

```jsx
const Cite = ({ documentKey, page, startText, endText }) => {
    const blob = await getDocumentBlob(documentKey); // Fetch the document blob from storage
    const url = URL.createObjectURL(blob);
    const iframeUrl = `${url}#page=${page}&:~:text=${encodeURIComponent(startText)},${encodeURIComponent(endText)}`;
    return (
        <Sheet>
            <iframe src={iframeUrl} />
        </Sheet>
    );
};
```

The clever part here is to use the Text Fragments feature of modern browsers to highlight the cited text in the PDF. This way, we don't need any libraries to parse and highlight the document. The Text Fragments API conveniently just asks for `textStart` and `textEnd`, which the LLM can provide easily.
Here is an example link to try it out: https://vincentmin.github.io/2025/11/11/generative-ui-chatbot-inline-citations-and-components.html#:~:text=here%20is%20an,cool%20is%20that
How cool is that?

These days most modern browsers support Text Fragments. As of recently, Google Chrome even supports Text Fragments for PDFs. This is really fantastic, as we can handle any other document type by converting it to PDF.
If the LLM or the Text Fragments API messes up the cited text, we fall back on the `#page=` parameter to at least open the document at the correct page.

Here is the application in action:
![Citations in action]({{ site.baseurl }}/assets/images/pdf-citations.png)

Clicking on the second citation badge opens the document at the correct page and highlights the cited text:
![Cited text highlighted]({{ site.baseurl }}/assets/images/pdf-sheet.png)

## Comparison to Anthropic's Citation API

Let's revisit the Anthropic Citation API for a moment. Their approach is similar in that the LLM sprinkles xml-like tags in its output. However, I prefer the approach outlined here for a few reasons:
- It is model provider agnostic. Even though OpenAI has yet to release a citation API, this approach works just as well with OpenAI as any other LLM provider.
- Anthropic first pre-process the documents by extracting text and prefixing the document and span id to each snippet. This is a lot of extra engineering work (that Anthropic kindly did for us), adds delay, and it bloats the prompt with many tokens.
- When using the Citation API, Anthropic injects specific instructions into the system prompt that we have no control over.
- The id prefixing approach relies on OCR/text extraction of the document. For images, there is no way to prefix the span ids. This is quite limiting. [Both Anthropic and OpenAI feed files to the LLM as both images and OCR-text](https://docs.claude.com/en/docs/build-with-claude/pdf-support#:~:text=Processes%20each%20page%20as%20both%20text%20and%20image%20for%20comprehensive%20understanding). Yet, [Gemini genuinely feeds only the images as text](https://ai.google.dev/gemini-api/docs/document-processing). This means that Google would need an entirely different approach to the one of Anthropic if they were to implement their own citation API.

Of course, Anthropic built an API and have no control over how users render the text. It makes total sense that they parse out the tags and leave it to the developer to render them. Because I'm assuming we have full control over both the backend and frontend, we can take a more lightweight approach.

## Conclusion

Mixing natural language generation with xml-like tags is a powerful technique to build rich user experiences with LLMs. The LLM sprinkles tags in its output, which are then parsed and rendered as actual components in the frontend. It unburdens the LLM from having to think about the rendering details, and allows developers to build custom components with standard software engineering practices.

Furthermore, we can make clever use of existing web platform features such as Text Fragments to implement powerful components like citations with minimal engineering effort.

You can define any component you like, so the possibilities are endless. Happy coding!
