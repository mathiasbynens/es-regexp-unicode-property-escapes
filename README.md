# ECMAScript proposal: Unicode property escapes in regular expressions

## Status

This proposal is in stage 0 of [the TC39 process](https://tc39.github.io/process-document/).

## Motivation

The Unicode Standard assigns various properties and property values to every symbol. For example, to get the set of symbols that are used exclusively in the Greek script, search the Unicode database for symbols whose `Script` property is set to `Greek`.

There currently is no way to access these Unicode character properties natively in ECMAScript regular expressions. This makes it painful for developers to support full Unicode in their regular expressions. They currently have two options, neither of which is ideal:

1. Use a library such as [XRegExp](https://github.com/slevithan/xregexp) to create the regular expressions at run-time:

    ```js
    const regexGreekSymbol = XRegExp('\\p{Greek}', 'A');
    regexGreekSymbol.test('π');
    // → true
    ```

    The downside of this approach is that the XRegExp library is a run-time dependency which may not be ideal for performance-sensitive applications. For usage on the web, there is an additional performance penalty: `xregexp-all-min.js.gz` takes up over 35 KB of space after minifying and applying gzip compression.

2. Use a library such as [Regenerate](https://github.com/mathiasbynens/regenerate) to generate the regular expression at build time:

    ```js
    const regenerate = require('regenerate');
    const codePoints = require('unicode-9.0.0/Script/Greek/code-points');
    const set = regenerate(codePoints);
    set.toString();
    // → '[\u0370-\u0373\u0375-\u0377\u037A-\u037D\u037F\u0384\u0386\u0388-\u038A\u038C\u038E-\u03A1\u03A3-\u03E1\u03F0-\u03FF\u1D26-\u1D2A\u1D5D-\u1D61\u1D66-\u1D6A\u1DBF\u1F00-\u1F15\u1F18-\u1F1D\u1F20-\u1F45\u1F48-\u1F4D\u1F50-\u1F57\u1F59\u1F5B\u1F5D\u1F5F-\u1F7D\u1F80-\u1FB4\u1FB6-\u1FC4\u1FC6-\u1FD3\u1FD6-\u1FDB\u1FDD-\u1FEF\u1FF2-\u1FF4\u1FF6-\u1FFE\u2126\uAB65]|\uD800[\uDD40-\uDD8E\uDDA0]|\uD834[\uDE00-\uDE45]'
    // Imagine there’s more code here to save this pattern to a file.
    ```

    This approach results in optimal run-time performance, although the generated regular expressions tend to be fairly large in size (which could lead to performance problems on the web). The biggest downside is that it requires a build script, which gets painful as the developer needs more Unicode-aware regular expressions.

## Proposed solution

We propose the addition of _Unicode property escapes_ of the form `\p{…}` and `\P{…}`. Unicode property escapes are a new type of escape sequence available in regular expressions that have the `u` flag set. With this feature, the above regular expression could be written as:

```js
const regexGreekSymbol = /\p{Script=Greek}/u;
regexGreekSymbol.test('π');
// → true
```

This proposal solves all the abovementioned problems:

* It is no longer painful to create Unicode-aware regular expressions.
* There is no dependency on run-time libraries.
* The regular expressions patterns are compact and readable — no more file size bloat.
* Creating a script that generates the regular expression at build time is no longer necessary.

## Illustrative examples

### Unicode-aware version of `\d`

To support any numeric symbol in Unicode rather than just ASCII `[0-9]`, use `\p{Number}` instead of `\d`.

```js
const regex = /^\p{Number}+$/u;
regex.test('²³¹¼½¾𝟏𝟐𝟑𝟜𝟝𝟞𝟩𝟪𝟫𝟬𝟭𝟮𝟯𝟺𝟻𝟼㉛㉜㉝');
// → true
```

### Unicode-aware version of `\w`

To support any word symbol in Unicode rather than just ASCII `[a-zA-Z0-9_]`, use `[\p{Letter}\p{Number}\p{Connector_Punctuation}\p{Mark}]`.

```js
const regex = /([\p{Letter}\p{Number}\p{Connector_Punctuation}\p{Mark}]+)/gu;
const text = `
Amharic: የኔ ማንዣበቢያ መኪና በዓሣዎች ተሞልቷል
Bengali: আমার হভারক্রাফ্ট কুঁচে মাছ-এ ভরা হয়ে গেছে
Georgian: ჩემი ხომალდი საჰაერო ბალიშზე სავსეა გველთევზებით
Macedonian: Моето летачко возило е полно со јагули
Vietnamese: Tàu cánh ngầm của tôi đầy lươn
`;

let match;
while (match = regex.exec(text)) {
  const word = match[1];
  console.log(`Matched word with length ${ word.length }: ${ word }`);
}
```

Console output:

```
Matched word with length 7: Amharic
Matched word with length 2: የኔ
Matched word with length 6: ማንዣበቢያ
Matched word with length 3: መኪና
Matched word with length 5: በዓሣዎች
Matched word with length 5: ተሞልቷል
Matched word with length 7: Bengali
Matched word with length 4: আমার
Matched word with length 11: হভারক্রাফ্ট
Matched word with length 5: কুঁচে
Matched word with length 3: মাছ
Matched word with length 1: এ
Matched word with length 3: ভরা
Matched word with length 3: হয়ে
Matched word with length 4: গেছে
Matched word with length 8: Georgian
Matched word with length 4: ჩემი
Matched word with length 7: ხომალდი
Matched word with length 7: საჰაერო
Matched word with length 7: ბალიშზე
Matched word with length 6: სავსეა
Matched word with length 12: გველთევზებით
Matched word with length 10: Macedonian
Matched word with length 5: Моето
Matched word with length 7: летачко
Matched word with length 6: возило
Matched word with length 1: е
Matched word with length 5: полно
Matched word with length 2: со
Matched word with length 6: јагули
Matched word with length 10: Vietnamese
Matched word with length 3: Tàu
Matched word with length 4: cánh
Matched word with length 4: ngầm
Matched word with length 3: của
Matched word with length 3: tôi
Matched word with length 3: đầy
Matched word with length 4: lươn
```

### Other examples

```js
const regex = /[\p{Letter}\p{Decimal_Number}]/u;
// Match all letters and decimal digits.
```

```js
const regexArrows = /^\p{Block=Arrows}+$/u;
regexArrows.test('←↑→↓↔↕↖↗↘↙↚↛↜↝↞↟↠↡↢↣↤↥↦↧↨↩↪↫↬↭↮↯↰↱↲↳↴↵↶↷↸↹↺↻↼↽↾↿⇀⇁⇂⇃⇄⇅⇆⇇⇈⇉⇊⇋⇌⇍⇎⇏⇐⇑⇒⇓⇔⇕⇖⇗⇘⇙⇚⇛⇜⇝⇞⇟⇠⇡⇢⇣⇤⇥⇦⇧⇨⇩⇪⇫⇬⇭⇮⇯⇰⇱⇲⇳⇴⇵⇶⇷⇸⇹⇺⇻⇼⇽⇾⇿');
// → true
```

```js
// ECMAScript parsers written in ECMAScript won’t need complex regular
// expressions anymore to match identifiers. Compare with e.g.
// https://gist.github.com/mathiasbynens/6334847.
const regexIdentifierStart = /[$_\p{ID_Start}]/u;
const regexIdentifierPart = /[$_\u200C\u200D\p{ID_Continue}\p{Other_ID_Start}]/u;
// Note: the following doesn’t account for reserved words in order to
// keep the example simple.
const regexIdentifier = /^(?:[$_\p{ID_Start}])(?:[$_\u200C\u200D\p{ID_Continue}\p{Other_ID_Start}])*$/u;
```
