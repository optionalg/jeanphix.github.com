---
layout: post
lang: en
title: Django real world functional testing
tags:
- django
- selenium
---

In the real world, a django application may be deployed on a real production plateform with as many layers as needed:

* WSGI server
* proxy
* ...

When we write functional tests with the [dummy web client](https://docs.djangoproject.com/en/dev/topics/testing/#test-client), we do not cover what's going on over all those layers that composed application globality.

Also, in the real world, we use web client intances such as browsers like chromium, opera, IE... to deal with the application over HTTP and we want our application to run correctly on all those clients.

Another important point is the use of client side javascripts: when we use the dummy client, we can't test javascripts.

That's why we need to write functionnal tests that cover real world constraints.

[Django 1.4](https://docs.djangoproject.com/en/dev/releases/1.4/) will bring us tools to do the job.

## Lets write a demo application

To simply illustrate this, I've made a simple (useless) application composed of a single user story:

    As an anonymous user, I can write a message, then the message is displayed in a list with the nine latest messages.

models.py:

{% highlight python %}
from django.db import models
from django.core.urlresolvers import reverse


class Post(models.Model):
    message = models.CharField(max_length=140)
    author = models.CharField(max_length=50)
    created_at = models.DateTimeField(auto_now=True, auto_now_add=True,
        db_index=True)

    def get_absolute_url(self):
        return "%s#%s" % (reverse('home'), str(self.id))
{% endhighlight %}

views.py:

{% highlight python %}
from django.views.generic.edit import CreateView
from models import Post
from settings import LATEST_POSTS_COUNT


class HomeView(CreateView):
    model = Post
    template_name = 'home.html'

    def get_context_data(self, **kwargs):
        context = super(HomeView, self).get_context_data(**kwargs)
        context['posts'] = Post.objects\
            .order_by('-created_at')[:LATEST_POSTS_COUNT]
        return context
{% endhighlight %}

Ok, now, I'll write a functional test case basically using the dummy client:

{% highlight python %}
from django.core.urlresolvers import reverse
from django.test import TestCase
from models import Post


class HomeTest(TestCase):
    def test_post_submission(self):
        initial_posts_count = Post.objects.count()
        r = self.client.post(reverse('home'), {
            'message': u'Hello world!',
            'author': u'jeanphix',
        }, follow=True)
        self.assertContains(r, '0 minutes ago')
        self.assertContains(r, 'Hello world!')
        self.assertEqual(initial_posts_count + 1, Post.objects.count())
{% endhighlight %}

Ok, the application now works correctly, but I think it's gonna be better to do the job ansynchronous. So, let's add a piece of javascript:

{% highlight javascript %}
window.addEvent('domready', function() {

    var form = document.id('message-form');
    form.addEvent('submit', function(e) {

        var request = new Request.JSON({
            url: form.action,

            onSuccess: function(data) {
                form.getElements('.errorlist').each(function(element) {
                    element.destroy();
                });
                if (data.errors) {
                    // Form is not valid
                    Object.each(data.errors, function(value, key) {
                        var input = document.id('id_' + key);
                        var previous = input.getParent('p').getPrevious();
                        var errorlist = new Element('ul', {
                            'class': 'errorlist',
                            html: '<li>' + value[0] + '</li>'
                        }).inject(input.getParent('p'), 'before');
                    });
                } else {
                    // New post has been created
                    var postList = document.id('post-list');
                    if (!postList) {
                        postList = new Element('ul', {'class': 'post-list'}).inject(form, 'after');
                    }
                    var li = new Element('li', {
                        id: data.id,
                        html: '0 minutes ago ' +
                        '<span class="author">' + data.author + '</span> wrote:' +
                        '<blockquote>' + data.message + '</blockquote>'
                    }).inject(postList, 'top');
                    form.getElement('textarea').set('value', '').focus();
                }
            }
        }).post(form);
        e.stop();
    });
});
{% endhighlight %}

and then modify views.py to feet my need:

{% highlight python %}
from django.views.generic.edit import CreateView
from django.http import HttpResponse
from django.utils import simplejson as json
from models import Post
from settings import LATEST_POSTS_COUNT


class HomeView(CreateView):
    model = Post
    template_name = 'home.html'

    def get_context_data(self, **kwargs):
        context = super(HomeView, self).get_context_data(**kwargs)
        context['posts'] = Post.objects\
            .order_by('-created_at')[:LATEST_POSTS_COUNT]
        return context

    def form_valid(self, form):
        resp = super(HomeView, self).form_valid(form)
        if self.request.is_ajax():
            data = {'id': self.object.id, 'message': self.object.message,
                'author': self.object.author}
            return self.get_json_response(data)
        else:
            return resp

    def form_invalid(self, form):
        if self.request.is_ajax():
            errors = {'errors': dict(form.errors)}
            return self.get_json_response(errors)
        else:
            return super(HomeView, self).form_invalid(form)

    def get_json_response(self, data):
        return HttpResponse(json.dumps(data), content_type='application/json')
{% endhighlight %}

Now my application works correctly even when javascript is disabled. Now it's gonna be cool to write a functionnal test to cover that.

## LiveServerTestCase

Django 1.4 introduced a new test class called [LiveServerTestCase](https://docs.djangoproject.com/en/dev/topics/testing/#live-test-server) that extends  TransactionTestCase.

This new test class launches the application in a real WSGI server instance into an asynchronous thread. As it's a real server, we can easily write tests within real web browser instances.

There are many solutions providing python backends ([Ghost.py](https://github.com/jeanphix/Ghost.py), [Selenium](http://readthedocs.org/docs/selenium-python/en/latest/)...), here I'm gonna use Selenium.

So, I had a new (minimalist) test case:


{% highlight python %}
from django.test import LiveServerTestCase
from models import Post

from selenium.webdriver.firefox.webdriver import WebDriver
from selenium.webdriver.support.ui import WebDriverWait


class BaseLiveTest(LiveServerTestCase):
    @classmethod
    def setUpClass(cls):
        cls.selenium = WebDriver()
        super(BaseLiveTest, cls).setUpClass()

    @classmethod
    def tearDownClass(cls):
        super(BaseLiveTest, cls).tearDownClass()
        cls.selenium.quit()


class LiveHomeTest(BaseLiveTest):
    def test_post_submission(self):
        initial_posts_count = Post.objects.count()
        self.selenium.get(self.live_server_url)
        message = self.selenium.find_element_by_name("message")
        message.send_keys('my message')
        self.selenium.find_element_by_name("author").send_keys('jeanphix')
        self.selenium.find_element_by_xpath('//input[@value="Send"]').click()
        WebDriverWait(self.selenium, 10).until(
            lambda x: self.selenium.find_element_by_css_selector('ul li'))
        self.assertEqual(initial_posts_count + 1, Post.objects.count())
{% endhighlight %}

As you can see, we can deal with database and browser in the same script...

That's all for now.
