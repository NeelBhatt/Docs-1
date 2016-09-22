---
uid: mvc/views/tag-helpers/authoring
---
# Authoring Tag Helpers

By [Rick Anderson](https://twitter.com/RickAndMSFT)

[View or download sample code](https://github.com/aspnet/Docs/tree/master/aspnet/mvc/views/tag-helpers/authoring/sample)

## Getting started with Tag Helpers

This tutorial provides an introduction to programming Tag Helpers. [Introduction to Tag Helpers](intro.md) describes the benefits that Tag Helpers provide.

A tag helper is any class that implements the `ITagHelper` interface. However, when you author a tag helper, you generally derive from `TagHelper`, doing so gives you access to the `Process` method. We will introduce the `TagHelper` methods and properties as we use them in this tutorial.

1. Create a new ASP.NET Core project called **AuthoringTagHelpers**. You won't need authentication for this project.

2. Create a folder to hold the Tag Helpers called *TagHelpers*. The *TagHelpers* folder is *not* required, but it is a reasonable convention. Now let's get started writing some simple tag helpers.

## Starting the email Tag Helper

In this section we will write a tag helper that updates an email tag. For example:

<!-- literal_block {"backrefs": [], "ids": [], "dupnames": [], "linenos": false, "names": [], "classes": [], "xml:space": "preserve", "language": "html", "highlight_args": {}} -->

````html

   <email>Support</email>
   ````

The server will use our email tag helper to convert that markup into the following:

<!-- literal_block {"backrefs": [], "ids": [], "dupnames": [], "linenos": false, "names": [], "classes": [], "xml:space": "preserve", "language": "html", "highlight_args": {}} -->

````html

   <a href="mailto:Support@contoso.com">Support@contoso.com</a>
   ````

That is, an anchor tag that makes this an email link. You might want to do this if you are writing a blog engine and need it to send email for marketing, support, and other contacts, all to the same domain.

1. Add the following `EmailTagHelper` class to the *TagHelpers* folder.

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/TagHelpers/z1EmailTagHelperCopy.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

````c#


   using Microsoft.AspNetCore.Razor.TagHelpers;
   using System.Threading.Tasks;

   namespace AuthoringTagHelpers.TagHelpers
   {
       public class EmailTagHelper : TagHelper
       {
           public override void Process(TagHelperContext context, TagHelperOutput output)
           {
               output.TagName = "a";    // Replaces <email> with <a> tag
           }
       }
   }
   ````

**Notes:**

* Tag helpers use a naming convention that targets elements of the root class name (minus the *TagHelper* portion of the class name). In this example, the root name of **Email**TagHelper is *email*, so the `<email>` tag will be targeted. This naming convention should work for most tag helpers, later on I'll show how to override it.

* The `EmailTagHelper` class derives from `TagHelper`. The `TagHelper` class provides methods and properties for writing Tag Helpers.

* The  overridden `Process` method controls what the tag helper does when executed. The `TagHelper` class also provides an asynchronous version (`ProcessAsync`) with the same parameters.

* The context parameter to `Process` (and `ProcessAsync`) contains information associated with the execution of the current HTML tag.

* The output parameter to `Process` (and `ProcessAsync`) contains a stateful HTML element representative of the original source used to generate an HTML tag and content.

* Our class name has a suffix of **TagHelper**, which is *not* required, but it's considered a best practice convention. You could declare the class as:

<!-- literal_block {"backrefs": [], "ids": [], "dupnames": [], "linenos": false, "names": [], "classes": [], "xml:space": "preserve", "language": "c#", "highlight_args": {}} -->

````c#

   public class Email : TagHelper
   ````

2. To make the `EmailTagHelper` class available to all our Razor views, we will add the `addTagHelper` directive to the *Views/_ViewImports.cshtml* file:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/Views/_ViewImportsCopyEmail.cshtml", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "html", "highlight_args": {"hl_lines": [2, 3], "linenostart": 1}} -->

````html

   @using AuthoringTagHelpers
   @addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
   @addTagHelper "AuthoringTagHelpers.TagHelpers.EmailTagHelper, AuthoringTagHelpers"
   ````

The code above uses the wildcard syntax to specify all the tag helpers in our assembly will be available. The first string after `@addTagHelper` specifies the tag helper to load (we are using "*" for all tag helpers), and the second string "AuthoringTagHelpers" specifies the assembly the tag helper is in. Also, note that the second line brings in the ASP.NET Core MVC tag helpers using the wildcard syntax (those helpers are discussed in [Introduction to Tag Helpers](intro.md).) It's the `@addTagHelper` directive that makes the tag helper available to the Razor view. Alternatively, you can provide the fully qualified name (FQN) of a tag helper as shown below:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/Views/_ViewImports.cshtml", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "html", "highlight_args": {"hl_lines": [3], "linenostart": 1}} -->

````html

   @using AuthoringTagHelpers
   @addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
   @addTagHelper "AuthoringTagHelpers.TagHelpers3.EmailTagHelper, AuthoringTagHelpers"

   ````

To add a tag helper to a view using a FQN, you first add the FQN (`AuthoringTagHelpers.TagHelpers.EmailTagHelper`), and then the assembly name (*AuthoringTagHelpers*). Most developers will prefer to use the wildcard syntax. [Introduction to Tag Helpers](intro.md) goes into detail on tag helper adding, removing, hierarchy, and wildcard syntax.

3. Update the markup in the *Views/Home/Contact.cshtml* file with these changes:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/Views/Home/Contact.cshtml", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "html", "highlight_args": {"hl_lines": [15, 16], "linenostart": 1}} -->

````html

   @{
       ViewData["Title"] = "Contact";
   }
   <h2>@ViewData["Title"].</h2>
   <h3>@ViewData["Message"]</h3>

   <address>
       One Microsoft Way<br />
       Redmond, WA 98052<br />
       <abbr title="Phone">P:</abbr>
       425.555.0100
   </address>

   <address>
       <strong>Support:</strong><email>Support</email><br />
       <strong>Marketing:</strong><email>Marketing</email>
   </address>

   ````

4. Run the app and use your favorite browser to view the HTML source so you can verify that the email tags are replaced with anchor markup (For example, `<a>Support</a>`). *Support* and *Marketing* are rendered as a links, but they don't have an `href` attribute to make them functional. We'll fix that in the next section.

> [!NOTE]
> Like [HTML tags and attributes](http://www.w3.org/TR/html-markup/documents.html#case-insensitivity), tags, class names and attributes in Razor, and C# are not case-sensitive.

## A working email Tag Helper

In this section, we'll update the `EmailTagHelper` so that it will create a valid anchor tag for email. We'll update it to take information from a Razor view (in the form of a `mail-to` attribute) and use that in generating the anchor.

Update the `EmailTagHelper` class with the following:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/TagHelpers/EmailTagHelperMailTo.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

````c#


   public class EmailTagHelper : TagHelper
   {
       private const string EmailDomain = "contoso.com";

       // Can be passed via <email mail-to="..." />. 
       // Pascal case gets translated into lower-kebab-case.
       public string MailTo { get; set; }

       public override void Process(TagHelperContext context, TagHelperOutput output)
       {
           output.TagName = "a";    // Replaces <email> with <a> tag

           var address = MailTo + "@" + EmailDomain;
           output.Attributes.SetAttribute("href", "mailto:" + address);
           output.Content.SetContent(address);
       }
   }



   ````

**Notes:**

* Pascal-cased class and property names for tag helpers are translated into their [lower kebab case](http://stackoverflow.com/questions/11273282/whats-the-name-for-dash-separated-case/12273101#12273101). Therefore, to use the `MailTo` attribute, you'll use `<email mail-to="value"/>` equivalent.

* The last line sets the completed content for our minimally functional tag helper.

* The highlighted line shows the syntax for adding attributes:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/TagHelpers/EmailTagHelperMailTo.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"hl_lines": [6], "linenostart": 1}} -->

````c#

   public override void Process(TagHelperContext context, TagHelperOutput output)
   {
       output.TagName = "a";    // Replaces <email> with <a> tag

       var address = MailTo + "@" + EmailDomain;
       output.Attributes.SetAttribute("href", "mailto:" + address);
       output.Content.SetContent(address);
   }

   ````

That approach works for the attribute "href" as long as it doesn't currently exist in the attributes collection. You can also use the `output.Attributes.Add` method to add a tag helper attribute to the end of the collection of tag attributes.

3. Update the markup in the *Views/Home/Contact.cshtml* file with these changes:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/Views/Home/ContactCopy.cshtml", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "html", "highlight_args": {"hl_lines": [15, 16], "linenostart": 1}} -->

````html

   @{
       ViewData["Title"] = "Contact Copy";
   }
   <h2>@ViewData["Title"].</h2>
   <h3>@ViewData["Message"]</h3>

   <address>
       One Microsoft Way Copy Version <br />
       Redmond, WA 98052-6399<br />
       <abbr title="Phone">P:</abbr>
       425.555.0100
   </address>

   <address>
       <strong>Support:</strong><email mail-to="Support"></email><br />
       <strong>Marketing:</strong><email mail-to="Marketing"></email>
   </address>
   ````

4. Run the app and verify that it generates the correct links.

**Note:** If you were to write the email tag self-closing (`<email mail-to="Rick" />`), the final output would also be self-closing. To enable the ability to write the tag with only a start tag (`<email mail-to="Rick">`) you must decorate the class with the following:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/TagHelpers/EmailTagHelperMailVoid.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"hl_lines": [1], "linenostart": 1}} -->

````c#

   [HtmlTargetElement("email", TagStructure = TagStructure.WithoutEndTag)] 
   public class EmailVoidTagHelper : TagHelper
   {
       private const string EmailDomain = "contoso.com";
       // Code removed for brevity

   ````

With a self-closing email tag helper, the output would be `<a href="mailto:Rick@contoso.com" />`. Self-closing anchor tags are not valid HTML, so you wouldn't want to create one, but you might want to create a tag helper that is self-closing. Tag helpers set the type of the `TagMode` property after reading a tag.

### An asynchronous email helper

In this section we'll write an asynchronous email helper.

1. Replace the `EmailTagHelper` class with the following code:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/TagHelpers/EmailTagHelper.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

````c#

   public class EmailTagHelper : TagHelper
   {
       private const string EmailDomain = "contoso.com";
       public override async Task ProcessAsync(TagHelperContext context, TagHelperOutput output)
       {
           output.TagName = "a";                                 // Replaces <email> with <a> tag
           var content = await output.GetChildContentAsync();
           var target = content.GetContent() + "@" + EmailDomain;
           output.Attributes.SetAttribute("href", "mailto:" + target);
           output.Content.SetContent(target);
       }
   }

   ````

**Notes:**

* This version uses the asynchronous `ProcessAsync` method. The asynchronous `GetChildContentAsync` returns a `Task` containing the `TagHelperContent`.

* We use the `output` parameter to get contents of the HTML element.

2. Make the following change to the *Views/Home/Contact.cshtml* file so the tag helper can get the target email.

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/Views/Home/Contact.cshtml", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "html", "highlight_args": {"hl_lines": [15, 16], "linenostart": 1}} -->

````html

   @{
       ViewData["Title"] = "Contact";
   }
   <h2>@ViewData["Title"].</h2>
   <h3>@ViewData["Message"]</h3>

   <address>
       One Microsoft Way<br />
       Redmond, WA 98052<br />
       <abbr title="Phone">P:</abbr>
       425.555.0100
   </address>

   <address>
       <strong>Support:</strong><email>Support</email><br />
       <strong>Marketing:</strong><email>Marketing</email>
   </address>

   ````

3. Run the app and verify that it generates valid email links.

## The bold Tag Helper

1. Add the following `BoldTagHelper` class to the *TagHelpers* folder.

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/TagHelpers/BoldTagHelper.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

````c#

   using Microsoft.AspNetCore.Razor.TagHelpers;

   namespace AuthoringTagHelpers.TagHelpers
   {
       [HtmlTargetElement(Attributes = "bold")]
       public class BoldTagHelper : TagHelper
       {
           public override void Process(TagHelperContext context, TagHelperOutput output)
           {
               output.Attributes.RemoveAll("bold");
               output.PreContent.SetHtmlContent("<strong>");
               output.PostContent.SetHtmlContent("</strong>");
           }
       }
   }

   /*
    * public IActionResult About()
   {
       ViewData["Message"] = "Your application description page.";
       return View("AboutBoldOnly");
       // return View();
   }
   */
   ````

**Notes:**

* The `[HtmlTargetElement]` attribute passes an attribute parameter that specifies that any HTML element that contains an HTML attribute named "bold" will match, and the `Process` override method in the class will run. In our sample, the `Process`  method removes the "bold" attribute and surrounds the containing markup with `<strong></strong>`.

* Because we don't want to replace the existing tag content, we must write the opening `<strong>` tag with the `PreContent.SetHtmlContent` method and the closing `</strong>` tag with the `PostContent.SetHtmlContent` method.

2. Modify the *About.cshtml* view to contain a `bold` attribute value. The completed code is shown below.

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/Views/Home/AboutBoldOnly.cshtml", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "html", "highlight_args": {"hl_lines": [7], "linenostart": 1}} -->

````html

   @{
       ViewData["Title"] = "About";
   }
   <h2>@ViewData["Title"].</h2>
   <h3>@ViewData["Message"]</h3>

   <p bold>Use this area to provide additional information.</p>

   <bold> Is this bold?</bold>
   ````

3. Run the app. You can use your favorite browser to inspect the source and verify the markup.

The `[HtmlTargetElement]` attribute above only targets HTML markup that provides an attribute name of "bold". The `<bold>` element was not modified by the tag helper.

4. Comment out the `[HtmlTargetElement]` attribute line and it will default to targeting `<bold>` tags, that is, HTML markup of the form `<bold>`. Remember, the default naming convention will match the class name **Bold**TagHelper to `<bold>` tags.

5. Run the app and verify that the `<bold>` tag is processed by the tag helper.

Decorating a class with multiple `[HtmlTargetElement]` attributes results in a logical-OR of the targets. For example, using the code below, a bold tag or a bold attribute will match.

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/TagHelpers/zBoldTagHelperCopy.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"hl_lines": [1, 2], "linenostart": 1}} -->

````c#

   [HtmlTargetElement("bold")]
   [HtmlTargetElement(Attributes = "bold")]
   public class BoldTagHelper : TagHelper
   {
       public override void Process(TagHelperContext context, TagHelperOutput output)
       {
           output.Attributes.RemoveAll("bold");
           output.PreContent.SetHtmlContent("<strong>");
           output.PostContent.SetHtmlContent("</strong>");
       }
   }

   ````

When multiple attributes are added to the same statement, the runtime treats them as a logical-AND. For example, in the code below, an HTML element must be named "bold" with an attribute named "bold" ( <bold bold /> ) to match.

<!-- literal_block {"backrefs": [], "ids": [], "dupnames": [], "linenos": false, "names": [], "classes": [], "xml:space": "preserve", "language": "c#", "highlight_args": {}} -->

````c#

   [HtmlTargetElement("bold", Attributes = "bold")]
   ````

You can also use the `[HtmlTargetElement]` to change the name of the targeted element. For example if you wanted the `BoldTagHelper` to target `<MyBold>` tags, you would use the following attribute:

<!-- literal_block {"backrefs": [], "ids": [], "dupnames": [], "linenos": false, "names": [], "classes": [], "xml:space": "preserve", "language": "c#", "highlight_args": {}} -->

````c#

   [HtmlTargetElement("MyBold")]
   ````

## Web site information Tag Helper

1. Add a *Models* folder.

2. Add the following `WebsiteContext` class to the *Models* folder:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/Models/WebsiteContext.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

````c#

   using System;

   namespace AuthoringTagHelpers.Models
   {
       public class WebsiteContext
       {
           public Version Version { get; set; }
           public int CopyrightYear { get; set; }
           public bool Approved { get; set; }
           public int TagsToShow { get; set; }
       }
   }
   ````

3. Add the following `WebsiteInformationTagHelper` class to the *TagHelpers* folder.

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/TagHelpers/WebsiteInformationTagHelper.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "none", "highlight_args": {"linenostart": 1}} -->

````none

   using System;
   using AuthoringTagHelpers.Models;
   using Microsoft.AspNetCore.Razor.TagHelpers;

   namespace AuthoringTagHelpers.TagHelpers
   {
       public class WebsiteInformationTagHelper : TagHelper
       {
           public WebsiteContext Info { get; set; }

         public override void Process(TagHelperContext context, TagHelperOutput output)
         {
            output.TagName = "section";
            output.Content.SetHtmlContent(
   $@"<ul><li><strong>Version:</strong> {Info.Version}</li>
   <li><strong>Copyright Year:</strong> {Info.CopyrightYear}</li>
   <li><strong>Approved:</strong> {Info.Approved}</li>
   <li><strong>Number of tags to show:</strong> {Info.TagsToShow}</li></ul>");
            output.TagMode = TagMode.StartTagAndEndTag;
         }
      }
   }
   ````

**Notes:**

* As mentioned previously, tag helpers translates Pascal-cased C# class names and properties for tag helpers into [lower kebab case](http://c2.com/cgi/wiki?KebabCase). Therefore, to use the `WebsiteInformationTagHelper` in Razor, you'll write `<website-information />`.

* We are not explicitly identifying the target element with the `[HtmlTargetElement]` attribute, so the default of `website-information` will be targeted. If you applied the following attribute (note it's not kebab case but matches the class name):

<!-- literal_block {"backrefs": [], "ids": [], "dupnames": [], "linenos": false, "names": [], "classes": [], "xml:space": "preserve", "language": "c#", "highlight_args": {}} -->

````c#

   [HtmlTargetElement("WebsiteInformation")]
   ````

The lower kebab case tag `<website-information />` would not match. If you want use the `[HtmlTargetElement]` attribute, you would use kebab case as shown below:

<!-- literal_block {"backrefs": [], "ids": [], "dupnames": [], "linenos": false, "names": [], "classes": [], "xml:space": "preserve", "language": "c#", "highlight_args": {}} -->

````c#

   [HtmlTargetElement("Website-Information")]
   ````

* Elements that are self-closing have no content. For this example, the Razor markup will use a self-closing tag, but the tag helper will be creating a [section](http://www.w3.org/TR/html5/sections.html#the-section-element) element (which is not self-closing and we are writing content inside the `section` element). Therefore, we need to set `TagMode` to `StartTagAndEndTag` to write output. Alternatively, you can comment out the line setting `TagMode` and write markup with a closing tag. (Example markup is provided later in this tutorial.)

* The `$` (dollar sign) in the following line uses an [interpolated string](https://msdn.microsoft.com/en-us/library/Dn961160.aspx):

<!-- literal_block {"backrefs": [], "ids": [], "dupnames": [], "linenos": false, "names": [], "classes": [], "xml:space": "preserve", "language": "html", "highlight_args": {}} -->

````html

   $@"<ul><li><strong>Version:</strong> {Info.Version}</li>
   ````

5. Add the following markup to the *About.cshtml* view. The highlighted markup displays the web site information.

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/Views/Home/About.cshtml", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "html", "highlight_args": {"hl_lines": [1, 12, 13, 14, 15, 16, 17, 18, 19, 20], "linenostart": 1}} -->

````html

   @using AuthoringTagHelpers.Models
   @{
       ViewData["Title"] = "About";
   }
   <h2>@ViewData["Title"].</h2>
   <h3>@ViewData["Message"]</h3>

   <p bold>Use this area to provide additional information.</p>

   <bold> Is this bold?</bold>

   <h3> web site info </h3>
   <website-information info="new WebsiteContext {
                                       Version = new Version(1, 3),
                                       CopyrightYear = 1638,
                                       Approved = true,
                                       TagsToShow = 131 }" />
   ````

**Note:** In the Razor markup shown below:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/Views/Home/About.cshtml", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "html", "highlight_args": {"linenostart": 1}} -->

````html

   <website-information info="new WebsiteContext {
                                       Version = new Version(1, 3),
                                       CopyrightYear = 1638,
                                       Approved = true,
                                       TagsToShow = 131 }" />
   ````

Razor knows the `info` attribute is a class, not a string, and you want to write C# code. Any non-string tag helper attribute should be written without the `@` character.

6. Run the app, and navigate to the About view to see the web site information.

**Note:**

* You can use the following markup with a closing tag and remove the line with `TagMode.StartTagAndEndTag` in the tag helper:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/Views/Home/AboutNotSelfClosing.cshtml", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "html", "highlight_args": {"linenostart": 1}} -->

````html

   <website-information info="new WebsiteContext {
                                       Version = new Version(1, 3),
                                       CopyrightYear = 1638,
                                       Approved = true,
                                       TagsToShow = 131 }" >
   </website-information>
   ````

## Condition Tag Helper

The condition tag helper renders output when passed a true value.

1. Add the following `ConditionTagHelper` class to the *TagHelpers* folder.

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/TagHelpers/ConditionTagHelper.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

````c#

   using Microsoft.AspNetCore.Razor.TagHelpers;

   namespace AuthoringTagHelpers.TagHelpers
   {
       [HtmlTargetElement(Attributes = nameof(Condition))]
       public class ConditionTagHelper : TagHelper
       {
           public bool Condition { get; set; }

           public override void Process(TagHelperContext context, TagHelperOutput output)
           {
               if (!Condition)
               {
                   output.SuppressOutput();
               }
           }
       }
   }

   ````

2. Replace the contents of the *Views/Home/Index.cshtml* file with the following markup:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/Views/Home/Index.cshtml", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "highlight_args": {"linenostart": 1}} -->

````

   @using AuthoringTagHelpers.Models
   @model WebsiteContext

   @{
       ViewData["Title"] = "Home Page";
   }

   <div>
       <h3>Information about our website (outdated):</h3>
       <Website-InforMation info=Model />
       <div condition="Model.Approved">
           <p>
               This website has <strong surround="em"> @Model.Approved </strong> been approved yet.
               Visit www.contoso.com for more information.
           </p>
       </div>
   </div>
   ````

3. Replace the `Index` method in the `Home` controller with the following code:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/Controllers/HomeController.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

````c#

     public IActionResult Index(bool approved = false)
     {
         return View(new WebsiteContext
         {
             Approved = approved,
             CopyrightYear = 2015,
             Version = new Version(1, 3, 3, 7),
             TagsToShow = 20
         });
     }

   ````

4. Run the app and browse to the home page. The markup in the conditional `div` will not be rendered. Append the query string `?approved=true` to the URL (for example, http://localhost:1235/Home/Index?approved=true). `approved` is set to true and the conditional markup will be displayed.

**Note:** We use the [nameof](https://msdn.microsoft.com/en-us/library/dn986596.aspx) operator to specify the attribute to target rather than specifying a string as we did with the bold tag helper:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/TagHelpers/zConditionTagHelperCopy.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"hl_lines": [1, 2, 5], "linenostart": 1}} -->

````c#

   [HtmlTargetElement(Attributes = nameof(Condition))]
    //   [HtmlTargetElement(Attributes = "condition")]
    public class ConditionTagHelper : TagHelper
   {
      public bool Condition { get; set; }

      public override void Process(TagHelperContext context, TagHelperOutput output)
      {
         if (!Condition)
         {
            output.SuppressOutput();
         }
      }
   }

   ````

The [nameof](https://msdn.microsoft.com/en-us/library/dn986596.aspx) operator will protect the code should it ever be refactored (we might want to change the name to `RedCondition`).

### Avoiding Tag Helper conflicts

In this section, we will write a pair of auto-linking tag helpers. The first will replace markup containing a URL starting with HTTP to an HTML anchor tag containing the same URL (and thus yielding a link to the URL). The second will do the same for a URL starting with WWW.

Because these two helpers are closely related and we may refactor them in the future, we'll keep them in the same file.

1. Add the following `AutoLinkerHttpTagHelper` class to the *TagHelpers* folder.

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/TagHelpers/z1AutoLinker.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

````c#

   [HtmlTargetElement("p")]
   public class AutoLinkerHttpTagHelper : TagHelper
   {
       public override async Task ProcessAsync(TagHelperContext context, TagHelperOutput output)
       {
           var childContent = await output.GetChildContentAsync();
           // Find Urls in the content and replace them with their anchor tag equivalent.
           output.Content.SetHtmlContent(Regex.Replace(
                childContent.GetContent(),
                @"\b(?:https?://)(\S+)\b",
                 "<a target=\"_blank\" href=\"$0\">$0</a>"));  // http link version}
       }
   }


   ````

**Notes:** The `AutoLinkerHttpTagHelper` class targets `p` elements and uses [Regex](https://msdn.microsoft.com/en-us/library/system.text.regularexpressions.regex.aspx) to create the anchor.

2. Add the following markup to the end of the *Views/Home/Contact.cshtml* file:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/Views/Home/Contact.cshtml", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "html", "highlight_args": {"hl_lines": [19], "linenostart": 1}} -->

````html

   @{
       ViewData["Title"] = "Contact";
   }
   <h2>@ViewData["Title"].</h2>
   <h3>@ViewData["Message"]</h3>

   <address>
       One Microsoft Way<br />
       Redmond, WA 98052<br />
       <abbr title="Phone">P:</abbr>
       425.555.0100
   </address>

   <address>
       <strong>Support:</strong><email>Support</email><br />
       <strong>Marketing:</strong><email>Marketing</email>
   </address>

   <p>Visit us at http://docs.asp.net or at www.microsoft.com</p>
   ````

3. Run the app and verify that the tag helper renders the anchor correctly.

4. Update the `AutoLinker` class to include the `AutoLinkerWwwTagHelper` which will convert www text to an anchor tag that also contains the original www text. The updated code is highlighted below:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/TagHelpers/z1AutoLinker.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"hl_lines": [15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34], "linenostart": 1}} -->

````c#

   [HtmlTargetElement("p")]
   public class AutoLinkerHttpTagHelper : TagHelper
   {
       public override async Task ProcessAsync(TagHelperContext context, TagHelperOutput output)
       {
           var childContent = await output.GetChildContentAsync();
           // Find Urls in the content and replace them with their anchor tag equivalent.
           output.Content.SetHtmlContent(Regex.Replace(
                childContent.GetContent(),
                @"\b(?:https?://)(\S+)\b",
                 "<a target=\"_blank\" href=\"$0\">$0</a>"));  // http link version}
       }
   }

   [HtmlTargetElement("p")]
   public class AutoLinkerWwwTagHelper : TagHelper
   {
       public override async Task ProcessAsync(TagHelperContext context, TagHelperOutput output)
       {
           var childContent = await output.GetChildContentAsync();
           // Find Urls in the content and replace them with their anchor tag equivalent.
           output.Content.SetHtmlContent(Regex.Replace(
               childContent.GetContent(),
                @"\b(www\.)(\S+)\b",
                "<a target=\"_blank\" href=\"http://$0\">$0</a>"));  // www version
       }
   }


   ````

5. Run the app. Notice the www text is rendered as a link but the HTTP text is not. If you put a break point in both classes, you can see that the HTTP tag helper class runs first. The problem is that the tag helper output is cached, and when the WWW tag helper is run, it overwrites the cached output from the HTTP tag helper. Later in the tutorial we'll see how to control the order that tag helpers run in. We'll fix the code with the following:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/TagHelpers/z1AutoLinkerCopy.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"hl_lines": [5, 6, 10, 21, 22, 26], "linenostart": 1}} -->

````c#

       public class AutoLinkerHttpTagHelper : TagHelper
       {
           public override async Task ProcessAsync(TagHelperContext context, TagHelperOutput output)
           {
               var childContent = output.Content.IsModified ? output.Content.GetContent() :
                   (await output.GetChildContentAsync()).GetContent();

               // Find Urls in the content and replace them with their anchor tag equivalent.
               output.Content.SetHtmlContent(Regex.Replace(
                    childContent,
                    @"\b(?:https?://)(\S+)\b",
                     "<a target=\"_blank\" href=\"$0\">$0</a>"));  // http link version}
           }
       }

       [HtmlTargetElement("p")]
       public class AutoLinkerWwwTagHelper : TagHelper
       {
           public override async Task ProcessAsync(TagHelperContext context, TagHelperOutput output)
           {
               var childContent = output.Content.IsModified ? output.Content.GetContent() : 
                   (await output.GetChildContentAsync()).GetContent();
     
               // Find Urls in the content and replace them with their anchor tag equivalent.
               output.Content.SetHtmlContent(Regex.Replace(
                    childContent,
                    @"\b(www\.)(\S+)\b",
                    "<a target=\"_blank\" href=\"http://$0\">$0</a>"));  // www version
           }
       }
   }

   ````

**Note:** In the first edition of the auto-linking tag helpers, we got the content of the target with the following code:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/TagHelpers/z1AutoLinker.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

````c#

   var childContent = await output.GetChildContentAsync();

   ````

That is, we call `GetChildContentAsync` using the `TagHelperOutput` passed into the `ProcessAsync` method. As mentioned previously, because the output is cached, the last tag helper to run wins. We fixed that problem with the following code:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/TagHelpers/z2AutoLinkerCopy.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"linenostart": 1}} -->

````c#

   var childContent = output.Content.IsModified ? output.Content.GetContent() :
       (await output.GetChildContentAsync()).GetContent();

   ````

The code above checks to see if the content has been modified, and if it has, it gets the content from the output buffer.

7. Run the app and verify that the two links work as expected. While it might appear our auto linker tag helper is correct and complete, it has a subtle problem. If the WWW tag helper runs first, the www links will not be correct. Update the code by adding the `Order` overload to control the order that the tag runs in. The `Order` property determines the execution order relative to other tag helpers targeting the same element. The default order value is zero and instances with lower values are executed first.

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/TagHelpers/z2AutoLinkerCopy.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"hl_lines": [5, 6, 7, 8], "linenostart": 1}} -->

````c#

   public class AutoLinkerHttpTagHelper : TagHelper
   {
       // This filter must run before the AutoLinkerWwwTagHelper as it searches and replaces http and 
       // the AutoLinkerWwwTagHelper adds http to the markup.
       public override int Order
       {
           get  {  return int.MinValue;   }
       }

   ````

The above code will guarantee that the HTTP tag helper runs before the WWW tag helper. Change `Order` to `MaxValue` and verify that the markup generated for the  WWW tag is incorrect.

## Inspecting and retrieving child content

The tag-helpers provide several properties to retrieve content.

* The result of `GetChildContentAsync` can be appended to `output.Content`.

* You can inspect the result of `GetChildContentAsync` with `GetContent`.

* If you modify `output.Content`, the TagHelper body will not be executed or rendered unless you call `GetChildContentAsync` as in our auto-linker sample:

<!-- literal_block {"xml:space": "preserve", "backrefs": [], "source": "/Users/shirhatti/docs/Docs/aspnet/mvc/views/tag-helpers/authoring/sample/AuthoringTagHelpers/src/AuthoringTagHelpers/TagHelpers/z1AutoLinkerCopy.cs", "ids": [], "dupnames": [], "names": [], "classes": [], "linenos": false, "language": "c#", "highlight_args": {"hl_lines": [5, 6, 10], "linenostart": 1}} -->

````c#

   public class AutoLinkerHttpTagHelper : TagHelper
   {
       public override async Task ProcessAsync(TagHelperContext context, TagHelperOutput output)
       {
           var childContent = output.Content.IsModified ? output.Content.GetContent() :
               (await output.GetChildContentAsync()).GetContent();

           // Find Urls in the content and replace them with their anchor tag equivalent.
           output.Content.SetHtmlContent(Regex.Replace(
                childContent,
                @"\b(?:https?://)(\S+)\b",
                 "<a target=\"_blank\" href=\"$0\">$0</a>"));  // http link version}
       }
   }

   ````

* Multiple calls to `GetChildContentAsync` will return the same value and will not re-execute the `TagHelper` body unless you pass in a false parameter indicating  not use the cached result.
