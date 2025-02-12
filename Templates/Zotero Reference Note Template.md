{#- infer latest annotation Date -#}
{%- macro maxAnnotationsDate() -%}
   {%- set tempDate = "" -%}
	{%- for a in annotations -%}
		{%- set testDate = a.date | format("YYYY-MM-DD#HH:mm:ss") -%}
		{%- if testDate > tempDate or tempDate == ""-%}
			{%- set tempDate = testDate -%}
		{%- endif -%}
	{%- endfor -%}
	{{tempDate}}
{%- endmacro -%}

{%- set colorCategoryToMeaning = {
"yellow": "Important",
"red": "Caveats, Drawbacks, Limitations",
"green": "Important to me",
"blue": "Reference, Term to lookup later",
"purple": "Method, Steps, Setting",
"magenta": "Results, Findings",
"orange": "Focus",
"gray": "Examples"
}-%}

{# lookup Zotero colors in annotations with Category #}
{%- macro getMeaning(colorCategory) -%}
	{%- if colorCategory-%}
		{{- colorCategoryToMeaning[colorCategory] -}}
	{%- else -%}
		{{- colorCategoryToMeaning["yellow"] -}}
	{%- endif -%}
{%- endmacro -%}

{%- set calloutHeaders = {
"highlight": "Highlight",
"strike": "Strike Through",
"underline": "Underline",
"note": "Sticky Note",
"image": "Image"
}-%}

{# lookup callout headers by type of annotation #}
{%- macro calloutHeader(type) -%}
	{%- if calloutHeaders[type]-%}
		{{- calloutHeaders[type] -}}
	{%- else -%}
		{{Note}}
	{%- endif -%}
{%- endmacro -%}

{#- handle space characters in zotero tags -#}
{%- macro printTags(rawTags) -%}
	{%- if rawTags.length > 0 -%}
		{%- for tag in rawTags -%}
			#zotero/{{ tag.tag | lower | replace(" ","_") }} {{ ' ' }} 
		{%- endfor -%}
	{%- endif %}
{%- endmacro -%}

{%- set inline_fields = {
"abstract": abstractNote,
"pdf": pdfZoteroLink, 
"extra": '"' ~ extra ~ '"',
"bibliography": '"' ~ bibliography ~ '"'
}
-%}

{%- set frontmatter_fields = {
"title": '"' ~ (title | replace ('"','') or caseTitle | replace ('"','')) ~ '"',
"authors": '[' ~ authors | replace (";", ", ") ~ ']',
"editors": '[' ~ editors | replace (";", ", ") ~ ']',
"directors": '[' ~ directors | replace (";", ", ") ~ ']',
"podcasters": '[' ~ podcasters | replace (";", ", ") ~ ']',
"scriptwriters": '[' ~ scriptwriters | replace (";", ", ") ~ ']',
"first-entry": minAnnotationsDate,
"last-entry": maxAnnotationsDate,
"online-uri": uri,
"added-to-zotero-at": dateAdded | format("YYYY-MM-DDTHH:mm:ss"),
"citekey": citekey,
"pages": numPages,
"running-time": runningTime,
"type": type,
"medium": itemType,
"library-catalog": libraryCatalog,
"pages": pages,
"language": language,
"url": url,
"isbn": ISBN
} -%}
{# generate field safely -#}
{%- macro generateField(prefix, delimiter, f, p) -%}
{%- if p and p != "[undefined]"-%}
{{prefix}}{{f}}{{delimiter}}{{p}}
{% endif %}
{%- endmacro -%}

{#- generate fields based on Zotero properties -#}
{%- macro generateFields(prefix, delimiter, fields) -%}
{%- for field, property in fields -%}
{%- if property.length > 0 -%}
{{- generateField(prefix, delimiter, field, property) -}}
{%- endif -%}
{%- endfor -%}
{%- endmacro -%}

---
aliases: ["{{title | replace ('"','')}}"{%- if authors and date-%}, "
{%- for author in authors -%}
{{author}}
{%- endfor -%}
{{" ("+date | format("YYYY") +") "}}{{title | replace ('"','')}}{{caseTitle | replace ('"','')}}"{%- endif -%}]
{{generateFields("",": ",frontmatter_fields) -}}
last-imported-to-obsidian: "{{importDate | format("YYYY-MM-DDTHH:mm:ss")}}"
{{""}}
{%- if date -%}
year-published: "{{date | format("YYYY")}}"
{%- endif -%}
{{""}}
---
{%- if ISBN -%}
{#![|200](https://covers.openlibrary.org/b/isbn/{{ISBN | replace ("-","")}}-M.jpg)#}
{%- endif -%}
{{ "" }}

#reference_note {{printTags(tags)}}
{{ "" }}

{%- macro prettifyInlineFields(fields) -%}
{%- for field, property in fields -%}
    {%- if property.length > 0 -%}
    > - **{{field | capitalize}}**:  {{property | nl2br}}
    {% endif %}
{%- endfor -%}
{%- endmacro -%}

> [!info]- Metadata
{{prettifyInlineFields(inline_fields)}}
{% if relations.length > 0 -%}
> 
> > [!note]- References:  
> >
> > | title | proxy note | desktopURI |
> > | --- | --- | --- |
{%- for r in relations %}
> > | {{r.title | replace("|","❕")}} | [[@{{r.citekey}}]] | [Zotero Link]({{r.desktopURI}}) |
{%- endfor -%}
{{ "" }}
{%- endif %}
{{ "" }}
{#%%🔥🔥🔥everything above this line might change during an update 🔥🔥🔥 %%#}
{{ " " }}
{%- set newAnnotations = annotations | filterby("date", "dateafter", lastImportDate) -%}
{% if newAnnotations.length > 0 %}
🔽*Imported (Annotations) on {{importDate | format("YYYY-MM-DD#HH:mm:ss")}}*🔽

{# Print meaning of the Color> [!annotation-{{ colorCategory | lower}}] {{getMeaning(colorCategory | lower)}}#}
{% for colorCategory, meaning in colorCategoryToMeaning %}
{%- set filteredAnnotations = newAnnotations | filterby("colorCategory", "startswith", colorCategory) -%}
{% if filteredAnnotations.length > 0 %}
# {{getMeaning(colorCategory | lower)}} ^{{colorCategory}}
{% for annotation in filteredAnnotations %}
> [!annotation-{{ colorCategory | lower}}] "{{- annotation.annotatedText | nl2br -}}"{{" "}}{%- if annotation.pageLabel %}[(p. {{annotation.pageLabel}})]{%- else %}[(ref.)]{%- endif %}({%- if annotation.desktopURI %}{{annotation.desktopURI}}{%- else %}zotero://open-pdf/library/items/{{annotation.attachment.itemKey}}?annotation={{annotation.id}}{%- endif %})^{{annotation.id}}
> 
{%- if annotation.annotatedText.length > 0 %}  
> {#- "{{- annotation.annotatedText | nl2br -}}"#}
{%- endif %}
{%- if annotation.imageRelativePath %}
> ![[{{annotation.imageRelativePath}}|300]]
{%- endif %}
{%- if annotation.comment %} 
> → {{annotation.comment | nl2br }}
{%- endif %}
{%- if annotation.tags.length > 0 %} 
> {{printTags(annotation.tags)}}
{%- endif %}
> {#> > {{annotation.date | format("YYYY-MM-DD HH:mm")}}#}
> {# > >{%- if annotation.desktopURI %}#}
> 
> {#{%- endif %} #}
{% endfor %}
{% endif %}
{% endfor %}
{%- endif -%}