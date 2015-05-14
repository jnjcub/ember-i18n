## Ember-I18n

Internationalization for Ember

### NOTE

The following documentation is for v4.0, which has not yet been released.
For documentation on the most recent release, see
[the v3.1.1 README](https://github.com/jamesarosen/ember-i18n/blob/v3.1.1/README.md).

### Requirements

Ember-I18n v4 requires

 * Ember v1.10+
 * Ember-CLI
 * jQuery v1.7 - v2.x

### Installation

For ember-cli projects, run

```bash
$ ember install ember-i18n
```

For non-ember-cli projects, use
[v3.1.1](https://github.com/jamesarosen/ember-i18n/tree/v3.1.1) or earlier.

### Defining Translations

Put translation files in `app/locales/[locale]/translations.js`. Each file
should export an object. If the values are `Function`s, they will be
used as-is. If they are `String`s, they will first be compiled using
`util:i18n/compile-translation`. A default Handlebars-like implementation is
provided. See below for more information on overriding the compiler.

For example,

```js
export default {
  'user.edit.title': 'Edit User',
  'user.followers.title.one': 'One Follower',
  'user.followers.title.other': 'All {{count}} Followers',

  // nested objects work just like dotted keys
  'button': {
    'add_user': {
      'title': 'Add a user',
      'text': 'Add',
      'disabled': 'Saving...'
    }
  }
};
```

The `translations` generator will generate a new translations file for you:

```bash
$ ember generate translations es
```

#### Adding Translations at Runtime

If you have a translations API (so you can manage them across apps centrally
or so you can deliver only the translations you need), you can add new
translations at runtime via the `service:i18n`:

```js
this.i18n.addTranslations({
  'user.profile.gravatar.help': 'Manage your avatar at gravatar.com.'
});
```

### `i18n` Service

Many pieces of ember-i18n rely on the `service:i18n`. This service is automatically
injected into every Route, Controller and Component in your app. If you need it
elsewhere, you can register your own injection:

```js
// app/initializers/i18n.js

export default {
  name: 'i18n',
  initialize: function(app) {
    app.inject('model', 'i18n', 'service:i18n')
  }
};
```

or you can ask for the service on a case-by-case basis:

```js
// app/services/session.js

export default Ember.Object.extend({
  i18n: Ember.inject.service()
});
```

### Setting the Locale

Set the current locale on the `i18n` service:

```js
// app/routes/application.js

export default Ember.Route.extend({
  afterModel: function(user) {
    this.i18n.set('locale', user.get('locale'));
  }
});
```

**Note**: if you do this in an initializer and use FastBoot, you probably want to
use an [`instance-initializer`](http://emberjs.com/blog/2015/05/13/ember-1-12-released.html#toc_instance-initializers).

### `{{t}}` Helper

A simple translation:
```html
<h2>{{t "user.edit.title"}}</h2>
```
yields
```html
<h2>Edit User</h2>
```

A translation based on a bound key:
```html
<h2>{{t title_i18n_key}}</h2>
```
yields
```html
<h2>Add a user</h2>
```
if `component.title_i18n_key` is `'button.add_user.title'`. If
it subsequently changes to `'user.edit.title'`, the HTML will
become
```html
<h2>Edit User</h2>
```

A translation with interpolated values:
```html
<h2>{{t "user.followers.title" count="2"}}</h2>
```
yields
```html
<h2>All 2 Followers</h2>
```

Interpolated values can be bound:
```html
<h2>{{t "user.followers.title" count=user.followers.count}}</h2>
```
yields
```html
<h2>All 2 Followers</h2>
```
if `user.get('followers.count')` returns `2`.

### Translation Computed Property Macro

`ember-i18n/translation-macro` defines a computed property macro that
makes it easy to define translated computed properties. For example,

```js
import translation from "ember-i18n/translation-macro";

export default Ember.Component.extend({

  // A simple translation.
  title: translation("user.edit.title"),

  followersCount: 1,

  // A translation with interpolations. This computed property
  // depends on `count` and will send `{ count: this.get('count') }`
  // in to the translation.
  followersTitle: translation("user.followers.title", { count: "followersCount" })

});
```

The first argument is the translation key. The second is a hash where the keys
are interpolations in the translation and the values are paths to the values
relative to `this`.

The macro relies on `this.i18n` being the `service:i18n`. See "i18n Service,"
above, for more information on where it is available.

### `i18n.t`

If the macro doesn't work for your use-case, you can use the i18n service
directly:

```js
export default Ember.Component.extend({

  // The dependency on i18n.locale is important if you want the
  // translated value to be recomputed when the user changes their locale.
  title: Ember.computed('i18n.locale', 'user.isAdmin', function() {
    if (this.get('user.isAdmin')) {
      return this.i18n.t('admin.edit.title');
    } else {
      return this.i18n.t('user.edit.title');
    }
  })

});
```

#### Translate Properties On Any Object

The `ember-i18n/translateable-properties` mixin automatically translates
any property ending in `"Translation"`:

```js
import TranslateableProperties from "ember-i18n/translateable-properties";

export default Ember.Object.extend(TranslateableProperties, {
  labelTranslation: 'button.add_user.title'
});

```

This will cause the `label` proeprty to be "Add a user".

#### Translate Attributes In a Component

The `ember-i18n/translateable-attributes` mixin automatically translates
HTML attributes ending in `"Translation"`.

Mix it in to your Component:

```js
import TranslateableAttributes from "ember-i18n/translateable-attriubtes";

export default Ember.Component.extend(TranslateableAttributes, {
  tagName: 'span'
});
```

Then define translations in the template:

```html
{{#my-span titleTranslation="button.add_user.title">
  {{t "button.add_user.text"}}
{{/my-span}}
```
yields
```html
<span title="Add a user">
  Add
</span>
```

### Pluralization

Ember-i18n includes support for inflection based on a `count` interpolation.
Pluralization rules are based on the
[Unicode Common Locale Data Repository](http://cldr.unicode.org/).

Whenever you pass the `count` option to the `t` function, template will be pluralized:

```js
// app/locales/en/translations.js:

export default {
  'dog': {
    'one': 'a dog',
    'other': '{{count}} dogs'
  }
};

// Elsewhere:

i18n.t('dog', { count: 1 }); // a dog
i18n.t('dog', { count: 2 }); // 2 dogs
```

ember-i18n converts `1` to `.one` and `2` to `.other`. Depending on the
locale, there could be up to 6 plural forms used: 'zero', 'one', 'two',
'few', 'many', 'other'.

If you want to override the inflection rules for a locale, you can define
your own in `app/locales/[locale]/plurals.js`. For example, to add support
for `zero` to English:

```js
// app/locales/en/plurals.js:

export default function pluralForm(n) {
  if (n === 0) { return 'zero'; }
  if (n === 1) { return 'one'; }
  return 'other';
}
```

### Translation Compiler

ember-i18n includes a default compiler that acts mostly like (unbound) Handlebars.
It supports interpolations with dots in them. It treats interpolated values as
HTML-*unsafe* by default. You can get HTML-safe interpolations in two ways:

Mark the interpolated value as HTML-safe:
```js
seeUserMessage: Ember.computed('i18n.locale', 'user.id', function() {
  var userLink = '<a href="/users/' + user.get('id') + '">' + user.get('name') + '</a>';

  return this.i18n.t('info.see-other-user', {
    userLink: Ember.String.htmlSafe(userLink)
  });
})
```

Use triple-stache notation in the translation:

```js
export default {
  'info.see-other-user': "Please see {{{userLink}}}"
};
```

In general, the first method is preferred because it makes it more difficult
to accidentally introduce an XSS vulnerability.

### Missing translations

When `t` is called with a nonexistent key, it returns the result of calling
`util:i18n/missing-translation` with the key and the context as arguments. The
default behavior is to return "Missing translation: [key]", but you can
customize this by overriding the function. The below example spits out the
key along with the values of any arguments that were passed:

```js
// app/utils/i18n/missing-translation:

export default function(key, context) {
  var values = Object.keys(context).map(function(key) { return context[key]; });
  return key + ': ' + (values.join(', '));
}

// Elsewhere:

t('nothing.here', { arg1: 'foo', arg2: 'bar' });
// => "nothing.here: foo, bar"
```

When a missing translation is encountered, a `missing` event is also triggered
on the `i18n` service, with the key and the context as arguments. You can use this
to execute other missing-translation behavior such as logging the key somewhere.

```js
i18n.on('missing', function(key, context) {
  Ember.Logger.warn("Missing translation: " + key);
};
```
