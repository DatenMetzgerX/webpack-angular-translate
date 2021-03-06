# webpack-angular-translate

[![NPM](https://nodei.co/npm/webpack-angular-translate.png?downloads=true&downloadRank=true&stars=true)](https://nodei.co/npm/webpack-angular-translate/)

[![Build Status](https://travis-ci.org/MichaReiser/webpack-angular-translate.svg?branch=master)](https://travis-ci.org/MichaReiser/webpack-angular-translate)
[![Coverage Status](https://coveralls.io/repos/MichaReiser/webpack-angular-translate/badge.svg?branch=master&service=github)](https://coveralls.io/github/MichaReiser/webpack-angular-translate?branch=master)
[![Dependency Status](https://gemnasium.com/DatenMetgzerX/webpack-angular-translate.svg)](https://gemnasium.com/DatenMetgzerX/webpack-angular-translate)
[![npm version](https://badge.fury.io/js/webpack-angular-translate.svg)](http://badge.fury.io/js/webpack-angular-translate)

This plugin extracts the translation id's and default texts from angular-translate and writes them into a separate json file.
The json file can be used by a backend component to initialize all the used translations. The benefit of this approach is,
that the frontend developer can define all translations directly in the frontend code and doesn't need to modify any backend-code.

## Getting started

Install the plugin using npm:

```bash
npm install webpack-angular-translate
```

Configure the loader and the plugin in the webpack configuration.

```js
var WebPackAngularTranslate = require("webpack-angular-translate");
{
  ...
  module: {
    rules: [
      {
        test: /\.html$/,
        use: [
          {
            loader: "html-loader",
            options: {
              removeEmptyAttributes: false,
              attrs: []
            }
          },
          {
            loader: WebPackAngularTranslate.htmlLoader()
          }
        ]
      },
      {
        test: /\.js/,
        loader: WebPackAngularTranslate.jsLoader(),
        options: {
          parserOptions: {
            sourceType: "module"
          }
        }
      }
    ]
  },

  plugins: [new WebPackAngularTranslate.Plugin()]
}
```

The htmlLoader should be used as pre loader as it expects html as input (and not html embedded into js, what is the result of the _html-loader_).
The javascriptLoader can be used like any other loader (pre / post or normal loader). The loader requires that the input is valid javascript. It's possible to only use the javascript or the html loader. It's advised to only apply the loader to relevant files, as it requires an additional parsing step, which has an effect on the build performance.

The plugin accepts the following options in the constructor:

- fileName: The name of the file that contains all the translations, default `translations.json`

The loaders accepts the name of a loader that should be applied in advance. E.g. the js loader can be applied to the result of the typescript loader:

```js
{
    test: /\.ts$/,
    loader: WebPackAngularTranslate.jsLoader('ts-loader')
}
```

### Custom HTML Translation extractors

The htmlLoader supports registering custom HTML text extractors. The API of an extractor is:

```
export interface HtmlTranslationExtractor {
  (element: AngularElement, context: HtmlTranslationExtractionContext): void;
}

export interface AngularElement {
  tagName: string;
  attributes: Attribute[];
  texts: Text[];
  startPosition: number;
}

export interface HtmlTranslationExtractionContext {
  emitError(message: string, position: number): void;
  emitSuppressableError(message: string, position: number): void;
  registerTranslation(translation: {
    translationId: string;
    defaultText?: string;
    position: number;
  }): void;

  asHtml(): void;
}
```

The extractor receives an angular element with all its attributes and its direct text siblings as well as a context that can be used to either register a new translation or emit a warning/error.
The [translate-directive-translation-extractor.ts](src/html/translate-directive-translation-extractor.ts) contains an implementation of an extractor.
Custom extractors can be specified with the html loader:

```js
{
  loader: WebPackAngularTranslate.htmlLoader({
    translationExtractors: [(element, context) => { ... }]
  })
}
```

#### AngularI18nTranslationsExtractor

WebpackAngularTranslates provides the `angularI18nTranslationsExtractor` to support extractions of translations in applications using [angular](https://angular.io/).
It extracts translations from the `i18n` and `i18n-[attr]` directives, used by [Angular for Internationalization](https://angular.io/guide/i18n).

```js
{
     use: [{
         loader: WebPackAngularTranslate.htmlLoader(),
         options: {
             translationExtractors: [WebPackAngularTranslate.angularI18nTranslationsExtractor]
         }
     }]
}
```

Examples:

`<h1 i18n="@@A title">A title</h1>` results in a translation with `{id: "A title", defaultTranslation: "A title"}` 

`<p i18n="@@loan-intro-description-text">This is a very long text for the loan intro!</p>` results in a translation with `{id: "loan-intro-description-text", defaultTranslation: "This is a very long text for the loan intro!"}` 

`<img src=... title="My image title" i18n-title="@@MyImage" />` results in a translation with `{id: "MyImage", defaultTranslation: "My image title"}`

Note: The extraction will only work for labels with an explicitly provided `@@id` and default translation.

## Supported Expressions

### Directive

The directive is supported for static translations. Dynamic translations are not supported.

```html
<div translate> Translation-ID</div>
<translate> Translation-ID</translate>

<div translate="Translation-ID"></div>
<translate="Translation-ID"></translate>

<div translate translate-default="Default text"> Translation-ID</div>
<translate translate-default="Default text">Translation-ID</div>

<!-- extension to angular-translate, define default for attributes -->
<a translate translate-attr-title="TITLE-ID" translate-default-attr-title="Default for title attr" href="#"><i class="fa-home fa" /></a>
```

### Filter

Filters in Angular-Expression statements are supported when the value, to which the filter is applied to, is a literal and no other filter is applied before the `translate` filter.
The following examples are supported:

```html
<h1 title="{{ 'My title' | translate }}"></h1>
<h2>{{ 'My long translation' | translate | limitTo:20 }}</h2>

<span>{{ "4" | translate }} {{ "x" | translate }}</span>
```

Filters in `ng-bind` and other attributes are currently not supported. In the most scenarios `ng-bind` can be replaced with the `translate` directive or a filter can be applied directly.

### Service

The `$translate` service is supported for literals only. No dynamic translations are supported. It's required
that the `$translate` service is always called `$translate`.

The following examples are supported:

```js
$translate('Login');

this.$translate('Login');
_this.$translate('Login'); // also other names then _this

$translate.instant('Login');
this.$translate.instant('Login');

$translate('Login', ..., ..., 'Anmelden');
```

If the `$translate` service is used with an expression (e.g. variable), then compilation will fail and an error is emitted
to the console. The error is emitted to remind the developer that he is responsible to register the dynamic translation.
If the dynamic translations have been registered, then the error can be suppressed using a `/* suppress-dynamic-translation-error: true */`
comment in the block of the `$translate` call. This will suppress all errors for the current block.

## Register dynamic translations

If a dynamic translation is used, then the translation needs to be registered. To do so, the `i18n.registerTranslations({ translationId: defaultText })` function can be used. This might be helpful if the translation id is dynamically composed from a value of a domain object but the set of all possible combinations is limited. The registered translations are merged into the outputted json file. The function calls will be replaced by `(0);`. If UglifyJS is used for minification, then those calls will be removed entirely.

An alternative is `i18n.registerTranslation(translationId, defaultText?)`. This function is intended to be used when the translation id's are known in the javascript code but not known in the html file. Following an example where the title is dynamically set, depending if it is a new item or an existing one:

```html
<h2 translate suppress-dynamic-translation-error>{{editCtrl.title}}</h2>
```

The controller defines the id of the translation to use:

```js
function EditController(user) {
  // compiles to this.title = user.isNew() ? "NEW_USER" : "EDIT_USER";
  this.title = user.isNew()
    ? i18n.registerTranslation("NEW_USER", "New user")
    : i18n.registerTranslation("EDIT_USER", "Edit user");
}
```

The call to `i18n.registerTranslation` registers the translation id with the default text (optional). The result of the function is the id of the translation to use. This makes it possible to register translations inplace. Calls to `i18n.registerTranslation` compile to the passed in translation id (the function is not evaluated at runtime).

The `suppress-dynamic-translation-error` attribute can be defined on any element and will suppress any errors from the plugin for the attributed element and all it's child elements. This attribute is removed for non debugging builds.

## API

**`i18n.registerTranslation(translationId: string, defaultText?: string): string`**

Registers a translation with the given translation id and default text. If the default text is absent, then the translation id is used as default text.

Returns the passed in translation id. Can be used to pass to the translate service or can be bound to a translate directive.

**`i18n.registerTranslations({ translationId: defaultText } ): string[]`**

Registers a set of translations. Accepts a single object where the keys are used as translation ids and the value are used as default text.

Returns an array containing the passed in translation ids. The array can be passed to the translate service.
