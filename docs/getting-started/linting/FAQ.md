# Troubleshooting / FAQ

## Table of Contents

- [My linting feels really slow](#my-linting-feels-really-slow)
- [I get errors telling me "The file must be included in at least one of the projects provided"](#i-get-errors-telling-me-the-file-must-be-included-in-at-least-one-of-the-projects-provided)
- [I use a framework (like Vue) that requires custom file extensions, and I get errors like "You should add `parserOptions.extraFileExtensions` to your config"](#i-use-a-framework-like-vue-that-requires-custom-file-extensions-and-i-get-errors-like-you-should-add-parseroptionsextrafileextensions-to-your-config)
- [I am using a rule from ESLint core, and it doesn't work correctly with TypeScript code](#i-am-using-a-rule-from-eslint-core-and-it-doesnt-work-correctly-with-typescript-code)
- [One of my lint rules isn't working correctly on a pure JavaScript file](#one-of-my-lint-rules-isnt-working-correctly-on-a-pure-javascript-file)
- [TypeScript should be installed locally](#typescript-should-be-installed-locally)

---

## My linting feels really slow

As mentioned in the [type-aware linting doc](./TYPED_LINTING.md), if you're using type-aware linting, your lint times should be roughly the same as your build times.

If you're experiencing times much slower than that, then there are a few common culprits.

### Wide includes in your `tsconfig`

When using type-aware linting, you provide us with one or more tsconfigs. We then will pre-parse all files so that full and complete type information is available.

If you provide very wide globs in your `include` (like `**/*`), it can cause many more files than you expect to be included in this pre-parse. Additionally, if you provide no `include` in your tsconfig, then it is the same as providing the widest glob.

Wide globs can cause TypeScript to parse things like build artifacts, which can heavily impact performance. Always ensure you provide globs targeted at the folders you are specifically wanting to lint.

### `eslint-plugin-prettier`

This plugin surfaces prettier formatting problems at lint time, helping to ensure your code is always formatted. However this comes at a quite a large cost - in order to figure out if there is a difference, it has to do a prettier format on every file being linted. This means that each file will be parsed twice - once by ESLint, and once by Prettier. This can add up for large codebases.

Instead of using this plugin, we recommend using prettier's `--list-different` flag to detect if a file has not been correctly formatted. For example, our CI is setup to run the following command automatically, which blocks diffs that have not been formatted:

```bash
$ yarn prettier --list-different \"./**/*.{ts,js,json,md}\"
```

### `eslint-plugin-import`

This is another great plugin that we use ourselves in this project. However there are a few rules which can cause your lints to be really slow, because they cause the plugin to do its own parsing, and file tracking. This double parsing adds up for large codebases.

There are many rules that do single file static analysis, but we provide the following recommendations.

We recommend you do not use the following rules, as TypeScript provides the same checks as part of standard type checking:

- `import/named`
- `import/namespace`
- `import/default`
- `import/no-named-as-default-member`

The following rules do not have equivalent checks in TypeScript, so we recommend that you only run them at CI/push time, to lessen the local performance burden.

- `import/no-named-as-default`
- `import/no-cycle`
- `import/no-unused-modules`
- `import/no-deprecated`

### The `indent` / `@typescript-eslint/indent` rules

This rule helps ensure your codebase follows a consistent indentation pattern. However this involves a _lot_ of computations across every single token in a file. Across a large codebase, these can add up, and severely impact performance.

We recommend not using this rule, and instead using a tool like [`prettier`](https://www.npmjs.com/package/) to enforce a standardized formatting.

---

## I get errors telling me "The file must be included in at least one of the projects provided"

This error means that the file that's being linted is not included in any of the tsconfig files you provided us. A lot of the time this happens when users have test files or similar that are not included.

There are a couple of solutions to this, depending on what you want to achieve.

- If you **do not** want to lint the file:
  - Use [one of the options ESLint offers](https://eslint.org/docs/user-guide/configuring#ignoring-files-and-directories) to ignore files, like a `.eslintignore` file, or `ignorePatterns` config.
- If you **do** want to lint the file:
  - If you **do not** want to lint the file with [type-aware linting](./TYPED_LINTING.md):
    - Use [ESLint's `overrides` configuration](https://eslint.org/docs/user-guide/configuring#configuration-based-on-glob-patterns) to configure the file to not be parsed with type information.
  - If you **do** want to lint the file with [type-aware linting](./TYPED_LINTING.md):
    - Check the `include` option of each of the tsconfigs that you provide to `parserOptions.project` - you must ensure that all files match an `include` glob, or else our tooling will not be able to find it.
    - If your file shouldn't be a part of one of your existing tsconfigs (for example, it is a script/tool local to the repo), then consider creating a new tsconfig (we advise calling it `tsconfig.eslint.json`) in your project root which lists this file in its `include`.

---

## I use a framework (like Vue) that requires custom file extensions, and I get errors like "You should add `parserOptions.extraFileExtensions` to your config"

You can use `parserOptions.extraFileExtensions` to specify an array of non-TypeScript extensions to allow, for example:

```diff
 parserOptions: {
   tsconfigRootDir: __dirname,
   project: ['./tsconfig.json'],
+  extraFileExtensions: ['.vue'],
 },
```

---

## I am using a rule from ESLint core, and it doesn't work correctly with TypeScript code

This is a pretty common thing because TypeScript adds new features that ESLint doesn't know about.

The first step is to [check our list of "extension" rules here](../../../packages/eslint-plugin/README.md#extension-rules). An extension rule is simply a rule which extends the base ESLint rules to support TypeScript syntax. If you find it in there, give it a go to see if it works for you. You can configure it by disabling the base rule, and turning on the extension rule. Here's an example with the `semi` rule:

```json
{
  "rules": {
    "semi": "off",
    "@typescript-eslint/semi": "error"
  }
}
```

If you don't find an existing extension rule, or the extension rule doesn't work for your case, then you can go ahead and check our issues. [The contributing guide outlines the best way to raise an issue](../../../CONTRIBUTING.md#raising-issues).

---

## One of my lint rules isn't working correctly on a pure JavaScript file

This is to be expected - ESLint rules do not check file extensions on purpose, as it causes issues in environments that use non-standard extensions (for example, a `.vue` and a `.md` file can both contain TypeScript code to be linted).

If you have some pure JavaScript code that you do not want to apply certain lint rules to, then you can use [ESLint's `overrides` configuration](https://eslint.org/docs/user-guide/configuring#configuration-based-on-glob-patterns) to turn off certain rules, or even change the parser based on glob patterns.

---

## TypeScript should be installed locally

Make sure that you have installed TypeScript locally i.e. by using `npm install typescript`, not `npm install -g typescript`,
or by using `yarn add typescript`, not `yarn global add typescript`. See https://github.com/typescript-eslint/typescript-eslint/issues/2041 for more information.
