# Punycode and IDNA Converter

Convert internationalized domain names to and from Punycode and see where one typed name becomes two DNS names: Python's built-in idna codec turns faß.de into fass.de, while the WHATWG URL Standard that browsers follow turns it into xn--fa-hia.de.

**Live demo:** https://0xelitesystem.github.io/punycode-idna-converter/

## Use

1. Type a bare domain (`bücher.de`), a full URL (`https://user@bücher.de:8443/x`) or an A-label (`xn--bcher-kva.de`), or press one of the three examples: `faß.de`, `βόλος.com`, and `a` + ZWJ + `b.com`.
2. Read the verdict. It is exactly one of:
   - **SAME**: both readings produce the same DNS name.
   - **SPLIT**: this name reaches two different DNS names.
   - **ONE SIDE REFUSES**: one reading produces a name and the other is refused. `a` + U+200D + `b.com` is refused by the WHATWG parser (CheckJoiners) and becomes `ab.com` once the joiner is deleted.
   - **BOTH REFUSE**: your browser's parser refuses it either way.
3. Read the two columns. **WHATWG (nontransitional)** is what your browser's own URL parser returns. **Deviation-mapped (UTS 46 Transitional Processing, deprecated), same flags as the WHATWG column** is what a client that still applies the deviation mapping computes.
4. Read the per-label table: the A-label, the U-label from this page's own RFC 3492 decoder, every code point as `U+XXXX`, the octet count against 63, and flags.
5. Use the **Raw Punycode** pane to encode or decode a bare string with no `xn--` prefix and no IDNA rules, including the RFC 3492 Appendix A mixed-case annotation.

### Real cases this answers

- **A script and a browser disagree about a link.** If the verdict is SPLIT, send the A-label from the WHATWG column instead of the Unicode text. UTS 46 section 4 validates Punycode without mapping in both modes, so `xn--fu-hia.de` (fuß.de) stays `xn--fu-hia.de` on both sides.
- **Registering or configuring an IDN.** The URL parser does not enforce DNS length (VerifyDnsLength is off for URLs), so a label of 64 × `é` becomes a 70-character A-label that the browser accepts. The page flags any label over 63 octets and any name over 253 octets.
- **An `xn--` label from a log, a certificate or an email header.** The page decodes it and checks that it is a real A-label: ASCII only after `xn--`, valid Punycode, at least one non-ASCII code point, and an unchanged round trip through your browser's parser. `xn--8i7caa` decodes to three fullwidth letters (U+FF57) that the parser maps to `www`, so it is flagged. `xn--a` decodes to U+0080, which the parser refuses. `xn--` alone decodes to nothing.
- **Invisible characters.** The converter echoes your input on a "Your input" line with ZWJ, ZWNJ, soft hyphens, controls and other format characters drawn as visible `U+XXXX` badges, and lists each one with its position. IDNA mapping deletes some of them (a soft hyphen disappears from both columns) and refuses others, so the input line is where they stay visible. Input is only ever written with `textContent`, never as HTML.
- **Testing your own Punycode code.** The in-page self test runs all 19 RFC 3492 section 7.1 sample strings (A to S) four ways each, four decoder rejections, the UTS 46 Table 1 rows, the A-label checks, and a positive control that is built to fail, so a self test that silently stopped running is visible.

### The mixed-scripts flag is a heuristic

Each character is tested against `\p{Script_Extensions=...}` for Latin, Cyrillic, Greek, Armenian, Georgian, Hebrew, Arabic, Devanagari, Bengali, Tamil, Sinhala, Thai, Han, Hiragana, Katakana, Hangul, Cherokee and Ethiopic, with Common and Inherited treated as neutral. If the per-character sets in a label have an empty intersection, the label is flagged. This is not confusable detection. A single-script lookalike is not flagged: `а.com` with Cyrillic U+0430 gives `xn--80a.com` and is not mixed. Normal Japanese that mixes Han, Hiragana and Katakana will show as mixed.

## Why this exists

Converter pages usually give one answer. There are two, and which one you get depends on which software reads the name.

The WHATWG URL Standard ([domain parser ToASCII](https://url.spec.whatwg.org/#concept-domain-to-ascii), Last Updated 10 September 2026) runs UTS 46 ToASCII with CheckHyphens=beStrict, CheckBidi=true, CheckJoiners=true, UseSTD3ASCIIRules=beStrict, Transitional_Processing=false, VerifyDnsLength=beStrict and IgnoreInvalidPunycode=false. For URLs, beStrict is false.

[UTS 46](https://www.unicode.org/reports/tr46/) (Version 18.0.0, Revision 36, 2026-08-31) names four deviation characters in Table 1: U+00DF ß, U+03C2 ς, U+200D ZWJ and U+200C ZWNJ. Section 2 says the only difference between Transitional and Nontransitional Processing "is the handling of the four Deviation characters", and section 4 adds one more rule: under Transitional Processing, U+1E9E capital sharp s is replaced by "ss". The deviation-mapped column therefore applies exactly five replacements to the whole input before parsing it with the same parser: ß and ẞ to `ss`, ς to σ, ZWJ and ZWNJ deleted. Verified against [IdnaMappingTable 18.0.0](https://www.unicode.org/Public/18.0.0/idna/IdnaMappingTable.txt) and 17.0.0: U+1E9E is the only code point whose mapping produces a deviation character.

That column is not a current browser mode. UTS 46 marks conformance clause C1 "(deprecated)", section 1.3 says "the industry has fully transitioned to IDNA2008 behavior, and transitional processing has been deprecated", and section 2 says "All major implementations have switched to nontransitional processing of the four deviation characters." It is also not IDNA2003, and the page never labels it that way: Python's IDNA2003 codec and UTS 46 Transitional Processing disagree on other inputs (see below).

UTS 46 section 1.3.2 describes the risk with its own hypothetical: an IRI for `www.sparkasse-gießen.de` that one client maps to `www.sparkasse-giessen.de` and another converts to `www.xn--sparkasse-gieen-2ib.de`, two names that can lead to two different sites. This page shows you the two names for whatever you paste.

It is one HTML file, MIT licensed, with no network call of any kind.

### Both readings

The deviation-mapped column is not hypothetical. Python's standard library still computes it. Each line below was run on Python 3.14.7 on 2026-10-01; run it yourself to check.

```
python -c "print('faß.de'.encode('idna'))"
b'fass.de'

python -c "print('βόλος.com'.encode('idna'), 'ශ්\u200dරී.com'.encode('idna'), 'نامه\u200cای.com'.encode('idna'))"
b'xn--nxasmq6b.com' b'xn--10cl1a0b.com' b'xn--mgba3gch31f.com'

python -c "import http.client; c=http.client.HTTPConnection('faß.de'); c.putrequest('GET','/'); print(c._buffer[1])"
b'Host: fass.de'

python -c "print('⒈.com'.encode('idna'))"
b'1..com'
```

The first two commands reproduce the IDNA2003 column of UTS 46 Table 1 for all four rows, and that is the only equivalence this page claims between Python and the deviation-mapped column. The last command shows they are not the same algorithm: Python gives `1..com` for `⒈.com`, while UTS 46 processing, transitional or not, records an error for it, because U+2488 is `disallowed` in IdnaMappingTable 18.0.0. Paste `⒈.com` into the page to see your browser refuse it in both columns.

The Python documentation for the codecs module says `encodings.idna` "implements RFC 3490" and that "the socket module transparently converts Unicode host names to ACE". [RFC 5891](https://www.rfc-editor.org/rfc/rfc5891) (August 2010) obsoletes RFC 3490 and RFC 3491, per its header. A browser following WHATWG sends `faß.de` to `xn--fa-hia.de`. Each side follows its own published rule, and the split exists only at the handoff.

### Limits, stated

- The WHATWG column is whatever your browser returns. The page runs a live check on load (`new URL('http://faß.de/').hostname === 'xn--fa-hia.de'`) and shows a banner if your browser does not use nontransitional processing. It prints your user agent and never prints a Unicode or IDNA version for the browser, because a page cannot read one.
- Never treat `new URL()` acceptance as validity for `xn--` input. The WHATWG domain parser returns ASCII input lowercased regardless of the ToASCII outcome when beStrict is false, and engines differ, so the page validates every `xn--` label itself.
- Length limits: each label 1 to 63 octets ([RFC 1035](https://www.rfc-editor.org/rfc/rfc1035) section 2.3.4, "labels 63 octets or less"; UTS 46 section 4.2) and the whole name 1 to 253 octets, excluding a trailing root dot (UTS 46 section 4.2). These are shown as flags, because the browser does produce the name.
- IPv4 and IPv6 hosts are reported as "IP address, IDNA does not apply". Non-special schemes are refused with an explanation, because the URL parser does not apply IDNA to their hosts.

### Method

Punycode is implemented from [RFC 3492](https://www.rfc-editor.org/rfc/rfc3492) sections 5 and 6; no code is reproduced. Parameters from section 5: base 36, tmin 1, tmax 26, skew 38, damp 700, initial_bias 72, initial_n 128. Overflow is detected per section 6.4 with maxint = 0xFFFFFFFF (an unsigned 32-bit integer). After each insertion the decoder rejects values above U+10FFFF and surrogates U+D800 to U+DFFF. Decoder digits are case-insensitive, and the Appendix A mixed-case annotation is returned per code point. The codec works over code points, never UTF-16 units.

This is an independent project. It is not affiliated with or endorsed by the Unicode Consortium, the WHATWG, the IETF, or the Python Software Foundation. Their names identify the specifications and software being compared.

## Privacy

Everything runs in your browser. Nothing you type is uploaded, logged or stored. There is no analytics, no telemetry, no font fetch, no CDN and no network call of any kind, and the page carries a Content-Security-Policy with `connect-src 'none'`. Converting a name does not look it up in DNS. The only thing written to disk is your light or dark theme choice in `localStorage`. Open the file, disconnect from the network, and it still works.

## Run locally

```
git clone https://github.com/0xelitesystem/punycode-idna-converter.git
cd punycode-idna-converter
```

Open `index.html` in any modern browser, straight from disk. Or serve it:

```
python -m http.server 8000
```

Then visit http://localhost:8000/

## Build

There is no build. One `index.html` with inline CSS and inline JavaScript, no dependencies, no package manager, no toolchain.

## License

MIT. Copyright (c) 2026 0xelitesystem. See [LICENSE](LICENSE).

## Related

- [unicode-character-inspector](https://github.com/0xelitesystem/unicode-character-inspector) - what each code point in a label actually is.
- [url-parser-and-query-editor](https://github.com/0xelitesystem/url-parser-and-query-editor) - the rest of the URL around the host.
- [domain-and-dns-basics-reference](https://github.com/0xelitesystem/domain-and-dns-basics-reference) - how a name becomes a DNS lookup.
