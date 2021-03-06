============
Localization
============

Kitsune is localized with `gettext <http://www.gnu.org/software/gettext/>`_.
User-facing strings in the code or templates need to be marked for gettext
localization.

We use `Verbatim <http://localize.mozilla.org/>`_ to provide an easy interface
to localizing these files. Localizers are also free to download the PO files
and use whatever tool they are comfortable with.


Making Strings Localizable
==========================

Making strings in templates localizable is exceptionally easy. Making strings
in Python localizable is a little more complicated. The short answer, though,
is just wrap the string in ``_()``.


Interpolation
-------------

A string is often a combination of a fixed string and something changing, for
example, ``Welcome, James`` is a combination of the fixed part ``Welcome,``,
and the changing part ``James``. The naive solution is to localize the first
part and the follow it with the name::

    _('Welcome, ') + username

This is **wrong!**

In some locales, the word order may be different. Use Python string formatting
to interpolate the changing part into the string::

    _('Welcome, {name}').format(name=username)

Python gives you a lot of ways to interpolate strings. The best way is to use
Py3k formatting and kwargs. That's the clearest for localizers.

The worst way is to use ``%(label)s``, as localizers seem to have all manner
of trouble with it. Options like ``%s`` and ``{0}`` are somewhere in the
middle, and generally OK if it's clear from context what they will be.


Localization Comments
---------------------

Sometimes, it can help localizers to describe where a string comes from,
particularly if it can be difficult to find in the interface, or is not very
self-descriptive (e.g. very short strings). If you immediately precede the
string with a comment that starts ``L10n:``, the comment will be added to the
PO file, and visible to localizers.

Example::

    rev_data.append({
                'x': 1000 * int(time.mktime(rdate.timetuple())),
                # L10n: 'R' is the first letter of "Revision".
                'title': _('R', 'revision_heading'),
                'text': unicode(_('Revision %s')) % rev.created
                #'url': 'http://www.google.com/'  # Not supported yet
            })


Adding Context with msgctxt
---------------------------

Strings may be the same in English, but different in other languages. English,
for example, has no grammatical gender, and sometimes the noun and verb forms
of a word are identical.

To make it possible to localize these correctly, we can add "context" (known in
gettext as "msgctxt") to differentiate two otherwise identical strings.

For example, the string "Search" may be a noun or a verb in English. In a
heading, it may be considered a noun, but on a button, it may be a verb. It's
appropriate to add a context (like "button") to one of them.

Generally, we should only add context if we are sure the strings aren't used in
the same way, or if localizers ask us to.

Example::

    from tower import ugettext as _

    ...

    foo = _('Search', context='text for the search button on the form')


Plurals
-------

"You have 1 new messages" grates on discerning ears. Fortunately, gettext gives
us a way to fix that in English *and* other locales, the ``ngettext``
function::

    ngettext('singular', 'plural', count)

A more realistic example might be::

    ngettext('Found {count} result.',
             'Found {count} results',
             len(results)).format(count=len(results))

This method takes three arguments because English only needs three, i.e., zero
is considered "plural" for English. Other locales may have different plural
rules, and require different phrases for, say 0, 1, 2-3, 4-10, >10. That's
absolutely fine, and gettext makes it possible.


Strings in Templates
--------------------

When putting new text into a template, all you need to do is wrap it in a
``_()`` call::

    <h1>{{ _('Heading') }}</h1>

Adding context is easy, too::

    <h1>{{ _('Heading', 'context') }}</h1>

L10n comments need to be Jinja2 comments::

    {# L10n: Describes this heading #}
    <h1>{{ _('Heading') }}</h1>

Note that Jinja2 escapes all content output through ``{{ }}`` by default. To
put HTML in a string, you'll need to add the ``|safe`` filter::

    <h1>{{ _('Firefox <span>Help</span>')|safe }}</h1>

To interpolate, you should use one of two Jinja2 filters: ``|f()`` or, in some
cases, ``|fe()``. ``|f()`` has exactly the same arguments as
``u''.format()``::

    {{ _('Welcome, {name}!')|f(name=request.user.username) }}

The ``|fe()`` is exactly like the ``|f()`` filter, but escapes its arguments
before interpolating, then returns a "safe" object. Use it when the localized
string contains HTML::

    {{ _('Found <strong>{0}</strong> results.')|fe(num_results) }}

Note that you *do not need* to use ``|safe`` with ``|fe()``. Also note that
while it may look similar, the following is *not* safe::

    {{ _('Found <strong>{0}</strong> results.')|f(num_results)|safe }}

The ``ngettext`` function is also available::

    {{ ngettext('Found {0} result.',
                'Found {0} results.',
                num_results)|f(num_results) }}


Using ``{% trans %}`` Blocks for Long Strings
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

When a string is very long, i.e. long enough to make Github scroll sideways, it
should be line-broken and put in a ``{% trans %}`` block. ``{% trans %}``
blocks work like other block-level tags in Jinja2, except they cannot have
other tags, except strings, inside them.

The only thing that should be inside a ``{% trans %}`` block is printing a
string with ``{{ string }}``. These are defined in the opening ``{% trans %}``
tag::

    {% trans user=request.user.username %}
        Thanks for registering, {{ user }}! We're so...
        hope that you'll...
    {% trans %}


Strings in Python
-----------------

.. Note::

   Whenever you are adding a string in Python, ask yourself if it
   really needs to be there, or if it should be in the template. Keep
   logic and presentation separate!

Strings in Python are more complex for two reasons:

#. We need to make sure we're always using Unicode strings and the
   Unicode-friendly versions of the functions.

#. If you use the ``ugettext`` function in the wrong place, the string may end
   up in the wrong locale!

Here's how you might localize a string in a view::

    from tower import ugettext as _

    def my_view(request):
        if request.user.is_superuser:
            msg = _(u'Oh hi, staff!')
        else:
            msg = _(u'You are not staff!')

Interpolation is done through normal Python string formatting::

    msg = _(u'Oh, hi, {user}').format(user=request.user.username)

``ugettext`` supports context, too::

    msg = _('Search', 'context')

L10n comments are normal one-line Python comments::

    # L10n: A message to users.
    msg = _(u'Oh, hi there!')

If you need to use plurals, import the function ``ungettext`` from Tower::

    from tower import ungettext, ugettext as _

    n = len(results)
    msg = ungettext('Found {0} result', 'Found {0} results', n).format(n)


Lazily Translated Strings
^^^^^^^^^^^^^^^^^^^^^^^^^

You can use ``ugettext`` or ``ungettext`` only in views or functions called
from views. If the function will be evaluated when the module is loaded, then
the string may end up in English or the locale of the last request! (We're
tracking down that issue.)

Examples include strings in module-level code, arguments to functions in class
definitions, strings in functions called from outside the context of a view. To
localize these strings, you need to use the ``_lazy`` versions of the above
methods, ``ugettext_lazy`` and ``ungettext_lazy``. The result doesn't get
translated until it is evaluated as a string, for example by being output or
passed to ``unicode()``::

    from tower import ugettext_lazy as _lazy

    PAGE_TITLE = _lazy(u'Page Title')

``ugettext_lazy`` also supports context.

It is very important to pass Unicode objects to the ``_lazy`` versions of these
functions. Failure to do so results in significant issues when they are
evaluated as strings.

If you need to work with a lazily-translated string, you'll first need to
convert it to a ``unicode`` object::

    from tower import ugettext_lazy as _lazy

    WELCOME = _lazy(u'Welcome, %s')

    def my_view(request):
        # Fails:
        WELCOME % request.user.username

        # Works:
        unicode(WELCOME) % request.user.username


Strings in the database
-----------------------

There is some user generated content that needs to be localizable. For
example, karma titles can be created in the admin site and need to be
localized when displayed to users. A django management command is used
for this. The first step to making a model's field localizable is adding
it to ``DB_LOCALIZE`` in ``settings.py``::

    DB_LOCALIZE = {
        'karma': {
            'Title': {
                'attrs': ['name'],
                'comments': ['This is a karma title.'],
            }
        },
        'appname': {
            'ModelName': {
                'attrs': ['field_name'],
                'comments': ['Optional comments for localizers.'],
            }
        }
    }

Then, all you need to do is run the ``extract_db`` management command::

    $ python manage.py extract_db

*Be sure to have a recent database from production when running the command.*

By default, this will write all the strings to `apps/sumo/db_strings.py`
and they will get picked up during the normal string extraction (see below).


Getting the Localizations
=========================

Localizations are not stored in this repository, but are in Mozilla's SVN::

    http://svn.mozilla.org/projects/sumo/locales

You don't need the localization files for general development. However, if
you need them for something, they're pretty easy to get::

    $ cd kitsune
    $ svn checkout https://svn.mozilla.org/projects/sumo/locales locale

(Alternatively, you can do yourself a favor and use::

    $ git svn clone -r HEAD https://svn.mozilla.org/projects/sumo/locales locale

if you're a git fan.)


Updating the Localizations
==========================

When strings are added or updated, we need to update the templates and PO files
for localizers. This needs to be coordinated with someone who has rights to
update the data on `Verbatim <http://localize.mozilla.org/>`_. If you commit
new strings to SVN and they are not updated right away on Verbatim, there will
be big merging headaches.

Updating strings is pretty easy. Check out the localizations as above, then::

    $ python manage.py extract
    $ python manage.py merge

Congratulations! You've now updated the POT and PO files.

Sometimes this can leave a bunch of garbage files with ``.po~`` extensions. You
should delete these, never commit them::

    $ find . -name "*.po~" -delete


Adding a New Locale
-------------------

Say you wanted to add ``fa-IR``::

    $ mkdir -p locale/fa-IR/LC_MESSAGES
    $ python manage.py merge

Then add 'fa-IR' to SUMO_LANGUAGES in settings.py.

And finally, add a migration with::

    INSERT INTO `wiki_locale` (`locale`) VALUES ('fa-IR');

Done!


Compiling MO Files
==================

gettext is so fast for localization because it doesn't parse text files, it
reads a binary format. You can easily compile that binary file from the PO
files in the repository.

We don't store MO files in the repository because they need to change every
time the corresponding PO file changes, so it's silly and not worth it. They
are ignored by ``svn:ignore``, but please make sure you don't forcibly add them
to the repository.

There is a shell script to compile the MO files for you::

    $ ./locale/compile-mo.sh locale

Done!
