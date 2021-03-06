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
    <p>{% if pub.work-type == "preprint" %}<strong>Preprint </strong>{% endif %}{{ pub.ref-title}} ({{ pub.ref-year}}): <a href="{{ site.baseurl }}{{ pub.url }}">Abstract</a> {% if pub.pdf-link != "" %} - <a href="{{ site.baseurl }}/{{ pub.pdf-link}}"> Download </a> {% endif %} </p>
    <p>{{ pub.ref-authors | markdownify }}</p>
    <p><em>{{ pub.ref-journal}}</em> {{ pub.ref-vol}} <a href="https://doi.org/{{ pub.ref-doi}}">{{ pub.ref-doi}}</a></p>
    {% if pub.preprint-doi != "NA" and pub.work-type != "preprint" %}<p>Originally a <a href="https://doi.org/{{ pub.preprint-doi}}">preprint</a>.</p>{% endif %}
    <div class='altmetric-embed' data-badge-type='donut' data-doi="{{ pub.ref-doi}}" style="float: left;width: 100px;" data-hide-no-mentions="true"></div>
    <span class="__dimensions_badge_embed__" data-doi="{{ pub.ref-doi}}" data-style="small_circle" data-hide-zero-citations="true"></span>
    <br />
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
    <p> Why do cells communicate? (2020) </p>
    <p> <strong> Johnston MG </strong> </p>
    <p> <a href="https://www.jic.ac.uk/blog/multicellularity-how-and-why-cells-communicate/">John Innes Centre Communications Blog</a> reproduced <a href="{{ site.base-url}}/WhyDoCellsCommuicate/">here</a>.</p>
  </li>
  <li>
    <p> Plant Cell Connections (2020) </p>
    <p> <strong> Johnston MG </strong> </p>
    <p> <em> Biological Sciences Review </em> 32(4):22-25 </p>
  </li>
  <li>
    <p> International Biology Olympiad Theory Papers (2017) </p>
    <p> <a href="https://ibo2017.rsb.org.uk/organisation/committees.html"><strong> Johnston MG* </strong>, Hodgson J* </a></p>
    <p> <em> IBO 2017 </em></p>
    <p> <a href="https://www.ibo-info.org/en/info/papers.html">Papers</a> </p>
  </li>
</ol>


Correct as of 13/01/2021.
<script type='text/javascript' src='https://d1bxh8uas1mnw7.cloudfront.net/assets/embed.js'></script>
<script async src="https://badge.dimensions.ai/badge.js" charset="utf-8"></script>
