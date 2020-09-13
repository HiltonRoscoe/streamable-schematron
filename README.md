# Streaming extensions for Schematron

This document discusses the language extensions necessary to support streaming (as defined in [XSLT3]) in Schematron.

## Motivation

There are many XML documents that require semantic validation beyond what XML Schema (or similar) can provide. Schematron has established itself as the de-facto standard for writing semantic constraints against XML documents. However, because most Schematron implementations generate and execute XSLT at runtime, they are limited to the capabilities of that language.

For a time, this precluded the use of Schematron for validation of large (for some definition of large) documents, as XSLT2 and prior did not support an approach to processing that was whose memory usage was stable.

Streaming was introduced in XSLT3, and addresses many shortcomings around processing larger files. However, to enable streaming in Schematron, some modifications/extensions to the standard may be necessary.

## Approach

Ideally, we would support as many streaming features as possible while introducing as few concepts to the Schematron specification.

## Basic streaming

Basic streaming refers to the native functionality available in an XSLT3 processor when streaming is enabled. This is handled through setting a mode as streamable, e.g.

```xslt
<xsl:mode streamable="yes" />
```

*Among other things

### Analysis

Basic streaming is the lowest lift to support. It involves identifying parts of the Schematron implementation that may preclude the use of streaming, and rewriting those portions to accommodate the features/restrictions of streaming.

This enables `sch:assert`s that are motionless.

## Burst mode streaming

Burst-mode streaming is not formally defined in XSLT3. It is an feature a XSLT processor can provide to work around the single downward select restriction. This is done by copying an entire element into memory and processing that copy. This is handled via the XPath functions `copy-of()` or `snapshot()` (if ancestors are desired).

This allows `sch:asserts` to access nodes on the child axis.

### Analysis

The first approach is provide set a new attribute for each `sch:rule` such that the use of burst-mode can be enabled for each assert. Note that this does not enable downward selections for the rule's `match`.

```
<sch:rule burst-mode="copy-of|snapshot" burst-target="{xpath}" context="book">
```

#### Attribute `burst-mode`

`burst-mode` specifies the XSLT function to use. 

#### Attribute `burst-target`

(optional) Specifies the target of `burst-mode` operation, if not the current node.

## Capturing Accumulators

Capturing accumulators are a feature of the *Saxon* XSLT processor. They are very useful when you have reference data appear prior to the `context` of the `sch:rule` that you wish to compare in one or more of its `sch:assert`s.

### Analysis

In order to minimize complexity of the streaming extensions to the Schematron specification, the syntax for using a capturing accumulator should be as simple as possible. One approach is to create a new element, e.g.

```xslt
<sch:reference name="vRefData" context="refData" />
```

which could appear directly under `sch:schema`. The `name` attribute can then be used in any `burst-mode` rule, as a variable, e.g.

```
<sch:assert test="groundedNode = $vRefData/someNode">
```

#### Implementation

`reference` tags are converted to accumulators.

```xslt
<xsl:accumulator name="vRefData"
                    as="node()?"
                    initial-value="()"
                    streamable="yes">
    <xsl:accumulator-rule xmlns:saxon="http://saxon.sf.net/"
                        match="refData"
                        select="."
                        phase="end"
                        saxon:capture="yes"/>
</xsl:accumulator>
```   

A `xsl:variable` is introduced to the grounded portion of each generated burst-mode `sch:rule`.

```xslt
<xsl:variable name="refData" select="accumulator-after('vRefData') />
```

> The variable cannot be declared globally because the state of the accumulator may change as more of the streamed document is read.

Optionally, we could add a attribute to `sch:rule` like `burst-use-reference` which would allow users to control which variables get defined, somewhat akin to `use-accumulators` (XSLT3).

**Alternative Approach (2)**

Create a function in the Schematron implementation, such as `sch:ref()`, which would serve as a wrapper around `accumulator-after`.

**Alternative Approach (3)**

Extend `sch:let` instead of introducing `sch:reference`.

## Regular Accumulators

let someone else fill this out :-)
