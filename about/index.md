---
layout: page
type: about
---

{% if page.comments != false %}

    {% if site.comments_provider == 'gitalk' %}
        <div id="gitalk-container"></div>
        <script src="/assets/js/gitalk.min.js"></script>
        <script>
        var gitalk = new Gitalk({
            id: '{{ page.url }}',
            clientID: '{{ site.gitalk.clientID }}',
            clientSecret: '{{ site.gitalk.clientSecret }}',
            repo: '{{ site.gitalk.repo }}',
            owner: '{{ site.gitalk.owner }}',
            admin: ['{{ site.gitalk.owner }}'],
            labels: ['gitment'],
            perPage: 50,
        })
        gitalk.render('gitalk-container')
        </script>
    {% endif %}
{% endif %}
