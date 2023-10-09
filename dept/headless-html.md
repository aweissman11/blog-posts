## A Simple Strategy for Migrating Content from a Traditional CMS to a headless CMS without difficult and costly data munging

<p></p>

### The Problem

Moving from a traditional Content Management System to a Headless CMS, and need to convert a large number of articles in HTML to JSON/Rich Text data. These formats do not translate 1:1, so you would need a custom script to make that transformation.

### The Challenge

Even simple HTML does not translate 1:1 to rich text. Something like text that is bold, italic and has a hyperlink may be nested three levels deep, whereas rich text will simply apply three attributes to that text block. Writing a script to convert that HTML into Rich Text when there is NO custom CSS is a challenge all it's own. If the text includes any custom styles, or more complex semantic HTML, this becomes an enormous development task that can take weeks at a minimum or months if there is a greater level of complexity or a large number or entries with unique formatting.

Here's an example of what a simple transformation would look like:

#### HTML:

```html
<p>
  <b>
    <em>
      <a href="https://example.com">Example</a>
    </em>
  </b>
</p>
```

#### Rich Text:

```json
[
  {
    "type": "paragraph",
    "children": [
      {
        "type": "text",
        "text": "Example",
        "marks": [
          {
            "type": "bold"
          },
          {
            "type": "italic"
          },
          {
            "type": "link",
            "attrs": {
              "href": "https://example.com"
            }
          }
        ]
      }
    ]
  }
]
```

If you have custom class names included in your rich text, most, if not all, rich text converters available on NPM will remove those class names when converting the HTML to Rich Text, resulting in a loss of potentially necessary styling data as well as making it impossible to go back through and find those text blocks that need to be styled a certain way.

One common example of this is the use of block quotes and pull quotes. Some CMSs might use the html `blockquote` element properly; many of them do not. Even if the CMS properly implemented this HTML tag, if you are writing articles that differentiate between a pull quote and a block quote, then you would have needed custom class names to style those quotes accordingly. In this case, any default conversion script will immediately lose that difference, requiring a huge chunk of time to go through and visually compare the original article to the data entered in the new headless CMS. If you were doing this via script, then you potentially need to handle a wide array of versions of the quote. It's not as simple as using the right module in the new headless CMS, but also converting all the text inside of the block properly (remember our bold, nested link above).

While this is one example, some other ones I've run into are integrated graphs where the HTML is using data attributes (these may be lost or require regex to pull out the right values from them), images (and their custom aspect ratios applied by CSS classes), and HTML tables (also known as the bane of rich text's existence). Each of these elements will require a custom script to convert it from the HTML to whichever Rich Text format you're using in whichever Headless CMS platform you've chosen. In some cases, like for tables or graph integrations, this will vary from project to project and may require recursive checks (think anchor tags nested inside of tables with lots of different `colspan`'s) to figure out how to format this into your own flavor of rich text.

### The Solution

My favorite, lightly hacky solution for this issue is to simply migrate over the raw HTML into your new headless CMS and then apply some scoped CSS to the content when rendering the article. Use this approach for migrated articles and your fancy new rich text for all articles moving forward.

### The Details

The real trick to this solution is to use it for the old articles but not the new ones. The gist of this is to script over all the articles into the new headless cms into a read only text field that's only visible on these old articles. Then use your WYSIWYG editor for any new articles being created.

To do this, I'll create an additional field called something like `contentType` with two options: "html" and "richText". Then, as I'm migrating over the old articles, I'll set the value to `contentType: "html"`. In the new CMS, hide the `contentType` field for any new articles being created and default the value of it to "richText". That way, new articles won't even know that HTML is an option, but if a content editor needs to go back and work on an old article, they'll see that it was ported over as HTML. At this point, if they need to edit that old article, they have two options: reach out to a developer to figure out how to edit the HTML, or build the article from scratch. While this may sound needlessly difficult ("Why not just allow them to edit the HTML?" you ask...), this is very much on purpose. We really do not want raw HTML being passed to the site. It's rare that these old articles will need any editing at all, and when they do, the time spent converting them to the new format is worthwhile, as it's likely that any old articles that require an update are ones that are frequently visited and therefore should reflect the new format of the CMS.

On the frontend, where we render these articles, it's as simple as doing a check on the `contentType` field and then rendering as you would otherwise. If the `contentType` is "html", then we use `dangerouslySetInnerHTML` for React, `v-html` for Vue, or whatever frontend framework you're using.

#### [Sanity.io](https://www.sanity.io/docs/schema-types) Example

```js
{
  title: 'Article',
  name: 'article',
  type: 'document',
  fields: [
    {
      title: 'Title',
      name: 'title',
      type: 'string'
    },
    {
      title: 'Content Type',
      name: 'contentType',
      type: 'string',
      options: {
        list: [
          {title: 'HTML', value: 'html'},
          {title: 'Rich Text', value: 'richText'}
        ]
      },
      initialValue: 'richText',
      hidden: ({ parent }) => !parent?.html?.length // Hide this field if the `HTML` field is empty
    },
    {
      title: 'HTML',
      name: 'html',
      type: 'text',
      hidden: ({ parent }) => !parent?.html?.length // Hide this field if the `HTML` field is empty
      rows: 50 // Make it easier to see the HTML when quick scanning it
    },
    {
      title: 'Content',
      name: 'content',
      type: 'array',
      of: [{type: 'block'}] // This is the Sanity name for rich text
    }
  ]
}
```

#### React example:

```jsx
import React from 'react';
import { PortableText } from '@portabletext/react';

export default function HtmlOrRichText({ block }) {
  if (block.contentType === 'html') {
    return (
      <div
        className="utility-class-to-style-raw-html"
        dangerouslySetInnerHTML={{ __html: block.htmlContent }}
      />
    );
  }

  return (
    <PortableText
      value={block.richTextContent}
      components={{
        blockQuote: ({ value }) => (
          <blockquote className="block-quote-class">{value}</blockquote>
        ),
        pullQuote: ({ value }) => (
          <blockquote className="pull-quote-class">{value}</blockquote>
        ),
      }}
    />
  );
}
```

### The Cleanup

The biggest issue with this approach is that, unless you spend an insane amount of time writing super specific CSS selectors (in which case you probably should have just written the data migration script anyway), you're likely going to have to accept a simplified approach to your styling on these old articles. In _most_ cases, this is fine. Often, the majority of these old articles haven't been visited in a while, and it's more about the content they hold for SEO purposes, than it is ensuring that they perfectly adhere to the new branding guidelines. If this is unacceptable, then this approach is not for you.

Given this shortcoming, I recommend simply going through and updating the top performing articles into the new format. With most Headless CMS, this is a fairly painless process that takes at most 10 or so minutes per article (often much less), depending on the length and complexity of the article and the new format. Now you can be sure that the top performing articles are styled in the new format, while also feeling confident that none of the old content has been lost in the other articles that have been migrated over. In some cases, the other, lower performing articles can be converted one by one after launch of the new site. In other cases, it's perfectly fine to leave them as is.

### Gotchas & Notes

A few notes for anyone taking up this approach:

- Make sure you convert image URLs. If the images are hosted on the old CMS, the URL may be something like: `https://old_cms_url.your_site.com/images/some-random-string`. You'll need to port these images over to the new site as part of the migration. This would be necessary whether you port over the article as HTML or convert it to rich text, so you're doing this either way. For this process, just make sure you do a find all and replace of the `old_cms_url` and replace the full `src` with the URL from your new CDN.
- It's worth finding some libraries to help with the rendering of the HTML. Here's one that pairs with chakra-ui: https://www.npmjs.com/package/@nikolovlazar/chakra-ui-prose, but there are lots out there to help with this. Remember, the idea of this approach is to move quickly.
- Make sure you're not siloed on how this all works. It's important that other people have context for what you're doing and why you're doing this. Otherwise you are going to have go back and script the whole thing all over again.
- Be aggressive about collecting other data from the posts. Don't rely on this approach for things like title, related articles, or any other type of tracking/SEO data that you need for the article. That should all be ported into your new headless CMS just like it would otherwise. This is just for the CONTENT of the article to make it simpler to render it properly.
- Beware of script tags. If you had users jamming script tags into the old articles, make sure you double check them to be sure they are working, or yank them out and covert those articles manually (if this is tenable). If there are too many script tags to port over manually, then, once again, this process is not for you.
- Save your work. Track your script in a repo so you can see what changes have happened to it over time in case you did something that caused a data loss.
- Save your data. Even though the old data is likely to take up an enormous amount of space, find a way to hang onto it well through the site launch. Use an S3 bucket or something if you have to, but make sure you can go back and grab the original data in case something got lost in translation here.
