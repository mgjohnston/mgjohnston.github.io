---
layout: page
title: Publications
permalink: /publications/
---

[ORCiD](https://orcid.org/0000-0003-1141-6135) [Google Scholar](https://scholar.google.co.uk/citations?user=nliFYiAAAAAJ) 

### Academic Publications
<ol>
{% for pub in site.categories.publication %}
  {% if pub.work-type == "poster" or pub.work-type == "oral" %}
    {% continue %}
  {% endif %}
  <li>
    <p>{% if pub.work-type == "preprint" %}<strong>Preprint </strong>{% endif %}{{ pub.ref-title}} ({{ pub.ref-year}}): <a href="{{ site.baseurl }}{{ pub.url }}">Abstract</a> {% if pub.pdf-link != "" %} - <a href="{{ site.baseurl }}/{{ pub.pdf-link}}"> PDF </a> {% endif %} </p>
    <p>{{ pub.ref-authors | markdownify }}</p>
    <p><em>{{ pub.ref-journal}}</em> {{ pub.ref-vol}} <a href="https://doi.org/{{ pub.ref-doi}}">{{ pub.ref-doi}}</a></p>
    {% if pub.preprint-doi != "NA" and pub.work-type != "preprint" %}<p>Originally a <a href="https://doi.org/{{ pub.preprint-doi}}">preprint</a>.</p>{% endif %}
  </li>
{% endfor %}
</ol>

### Talks and posters
<ol>
{% for pub in site.categories.publication %}
  {% if pub.work-type == "paper" or pub.work-type == "preprint" %}
    {% continue %}
  {% endif %}
  <li>
    <p><strong>
      {% if pub.work-type == "poster" %}
        Poster: 
      {% elsif pub.work-type == "oral" %}
        Oral presentation: 
      {% endif %}
      </strong> {{ pub.ref-title}} ({{ pub.ref-year}})</p>
    <p>{{ pub.ref-authors | markdownify }}</p>
    <p><em>{{ pub.ref-journal}}</em></p>
  </li>
{% endfor %}
</ol>

### Other Publications
<ol>
  <li>
    <p> Cell-to-cell Communication (2020) </p>
    <p> <strong> Johnston MG </strong> </p>
    <p> <a href="#">John Innes Centre Communications Blog</a> reproduced <a href="{{ site.base-url}}/WhyDoCellsCommuicate/">here</a>.</p>
  </li>
  <li>
    <p> Plant Cell Connections (2020) </p>
    <p> <strong> Johnston MG </strong> </p>
    <p> <em> Biological Sciences Review </em> 32(4) </p>
  </li>
  <li>
    <p> International Biology Olympiad Theory Papers (2017) </p>
    <p> <a href="https://ibo2017.rsb.org.uk/organisation/committees.html"><strong> Johnston MG* </strong>, Hodgson J* </a></p>
    <p> <em> IBO 2017 </em></p>
    <p> <a href="https://www.ibo-info.org/en/info/papers.html">Papers</a> </p>
  </li>
</ol>


Correct as of 01/03/2020.
