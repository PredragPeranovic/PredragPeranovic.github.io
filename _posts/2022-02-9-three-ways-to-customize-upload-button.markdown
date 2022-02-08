---
layout: post
title:  "Three Ways To Customize HTML Upload Button"
date:   2022-02-09 00:09:00 +0100
categories: ["HTML", "How to"]
---

# 1. Call input.click() With Javascript

{% highlight html %}
<button type="button" onclick="document.getElementById('fileUpload').click()">
    Upload file
</button>
<input type="file" id="fileUpload" style="display:none">
{% endhighlight %}
{% highlight css %}
button {
  background-color: skyblue;
  border-radius: 4px;
  border: none;
  height: 32px
}
button:hover {
  background-color: darkblue;
  color: lightblue;
}
{% endhighlight %}

Demo at [codepen](https://codepen.io/peranp/pen/zYPZNoE)

#### Pros

- Full freedom to customize the button;
- Widely supported by all browsers;
- **I like it very much**;

#### Cons

- Javascript is used with HTML and CSS;
- File input must be uniquely selectable by Javascript;
- Default "input element" isn't displayed so you must add custom HTML to display uploaded/selected file;

# 2. Styling Label as a Button

{% highlight html %}
<label for="fileUpload" class="fileUploadLabel">Upload file</label>
<input type="file" id="fileUpload" style="display:none">
{% endhighlight %}
{% highlight css %}
.fileUploadLabel {
  display: flex;
  margin: auto;
  align-items: center;
  justify-content: center;
  border-radius: 4px;
  background-color: skyblue;
  border: none;
  height: 32px;
  width: fit-content;
  padding: 0 8px;
  cursor: pointer;
}
.fileUploadLabel:hover {
  background-color: darkblue;
  color: lightblue;
}
{% endhighlight %}

Demo at [codepen](https://codepen.io/peranp/pen/qBVrRbo?editors=1100).


#### Pros

- Full freedom to customize the button;
- It works with just HTML and CSS;
- Widely supported by all browsers;
- Default "input element" isn't displayed;

#### Cons
- **Upload "button" isn't focusable with keyboard**;
- Default "input element" isn't displayed so you must add custom HTML to display uploaded/selected file;


# 3. Use ::file-selector-button CSS pseudo element

{% highlight html %}
<label for="fileUpload">Upload file</label>
<input type="file" id="fileUpload">
{% endhighlight %}
{% highlight css %}
input[type="file"]::-webkit-file-upload-button, 
input[type="file"]::file-selector-button {
  border-radius: 4px;
  background-color: skyblue;
  border: none;
  height: 32px;
}

input[type="file"]::-webkit-file-upload-button:hover,
input[type="file"]::file-selector-button:hover{
  background-color: darkblue;
  color: lightblue;
}
{% endhighlight %}

Demo at [codepen](https://codepen.io/peranp/pen/xxPqRoL).

Read more at [mdn](https://developer.mozilla.org/en-US/docs/Web/CSS/::file-selector-button).

## Pros
- Part of CSS specification;

## Cons
- Part of "fragile", "unfinished" and not wiedly supported CSS specification;
- Limited browser support;
- Must use browser-specific pseudo-classes to support different browsers;
- Limited ability to style "input element"  next to the button;
- **Don't like it at all**;
