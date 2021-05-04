# autocomplete-vue

This is a Vue component for autocomplete, built with accessibility in mind. This repo includes a whole app, with an
example implementation.

Reference implementation is here: (vanilla JavaScript): https://github.com/realsuayip/autocomplete

### AutoComplete Component (events and props)

`@selected`\
Pass the method to be executed after an autocomplete result has been selected. An argument is provided, which correspond
to selected object, i.e., `{name, value}`.

`@lookup`\
Pass the method which will return the autocomplete results. `query` (the lookup keyword)
and `done` (callback function) are provided as arguments. Callback function requires `query` and
`results` arguments, results must be a list of `{name, value}`.

`no-results-message`\
The message to be shown when there are no results found. If not set, no message will be shown.

`input-id`\
Set this option if you require custom id attribute for the autocomplete input.

`highlight`\
Set this option to true for highlighting.

`cache`\
Set this option to true to enable caching.

### Project setup

```
npm install
```

#### Compiles and hot-reloads for development

```
npm run serve
```

#### Compiles and minifies for production

```
npm run build
```

#### Lints and fixes files

```
npm run lint
```

### License

MIT