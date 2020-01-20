---
layout: page
title: Publications
permalink: /publications/
---



[ORCiD](https://orcid.org/0000-0003-1141-6135) [Google Scholar](https://scholar.google.co.uk/citations?user=nliFYiAAAAAJ) 

### Academic Publications
<ol>
{% for pub in site.categories.publication %}
  <li>
    <p>{% if pub.work-type == "preprint" %} Preprint {% endif %}{{ pub.ref-title}} ({{ pub.ref-year}})</p>
    <p>{{ pub.ref-authors | markdownify }}</p>
    <p><em>{{ pub.ref-journal}}</em> {{ pub.ref-vol}} <a href="https://doi.org/{{ pub.ref-doi}}">{{ pub.ref-doi}}</a></p>
    {% if preprint-doi != "NA" And  pub.work-type != "preprint" %}<p>Originally a [preprint](https://doi.org/{{ pub.preprint-doi }})</p>{% endif %}
  </li>
{% endfor %}
</ol>
### Other Publications

1. **Plant Cell Connections**
**Johnston MG**
_Biological Sciences Review_, 32 4, April 2020

Correct as of 20/01/2020.
