# Implementation of Streamable Schematron

This document describes methods of implementing this proposal, using `schxslt`.

## Handling `streaming` attribute of `sch:rule`

### `off`

When streaming is turned off (the default), rules are converted to XSLT stuctures as before, perhaps with some optimizations taking advantage of XSLT3.

### `on`

When streaming is turned `on`, the rules are converted to XSLT structures as before, but using a streaming `mode`.

### `copy-of`, `snapshot`

When streaming is turned to `copy-of` or `snapshot`, a streaming wrapper `xsl:template` is generated, which runs `xsl:apply-templates` using the `copy-of` or `snapshot` operation against a grounded version of the rule template. Additionally, templates generated from `streaming="on"` rules are run.

### `inherit`

When streaming is turned to `inherit`, rules are converted to XSLT structures as before (i.e. in a non-streaming mode).

> There is little practical difference between `off` and `inherit`. It mostly signals that the rule *wants* to use bursting. There may be some design time or compile time analysis that this enables.

## Handling of `sch:reference`

TODO: Fill this in.