# Streaming extensions for Schematron

This document discusses the language extensions necessary to support streaming (as defined in [XSLT3]) in Schematron.

## Motivation

There are many XML documents that require semantic validation beyond what XML Schema (or similar) can provide. Schematron has established itself as the de facto standard for writing semantic constraints against XML documents. However, because most Schematron implementations use XSLT as their runtime engine, they are limited to the capabilities of that language.

For a time, this precluded the use of Schematron for validation of large (for some definition of large) documents, as XSLT2 and prior did not support an approach to processing whose memory usage was stable.

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

Basic streaming is the lowest lift to support. It involves identifying parts of a Schematron implementation that may preclude the use of streaming, and rewriting those portions to accommodate the features/restrictions of streaming.

This enables `sch:assert`s that are motionless.

## Burst mode streaming

Burst-mode streaming is not formally defined in XSLT3. It is an feature an XSLT processor can provide to work around the single downward select restriction. This is done by copying an entire element into memory and processing that copy. This is handled via the XPath functions `copy-of()` or `snapshot()` (if ancestor access is desired).

This enables `sch:assert`s to access nodes on the child axis.

### Analysis

The first approach is provide set a new attribute for each `sch:rule` such that the use of burst-mode can be enabled for each assert. 

```xml
<sch:rule streaming="on|off|copy-of|snapshot|inherit" context="book">
```

> Note that this does not enable downward selections for the `sch:rule`'s `match`.

#### Attribute `streaming`

`streaming` specifies the XSLT function to use. 

## Capturing Accumulators

Capturing accumulators are a feature of the *Saxon* XSLT processor. They are very useful when you have reference data appearing prior to the `context` of the `sch:rule` that you wish to use in one or more of `sch:assert`s.

### Analysis

In order to minimize complexity of the streaming extensions to the Schematron specification, the syntax for using a capturing accumulator should be as simple as possible. One approach is to create a new element, e.g.

```xslt
<sch:reference name="vRefData" context="refData" select="." />
```

which could appear directly under `sch:schema`. The `name` attribute can then be used in any `burst-mode` rule, as a variable, e.g.

```xml
<sch:assert test="groundedNode = $vRefData/someNode">
```

#### Attribute `apply-rules`

This will cause rules to be run in a grounded context, as the streaming rules will have already run. This can possibly save memory, by avoiding writing `sch:rule`s that use `streaming="copy-of|snapshot"` to validate captured data. 

#### Attribute `context`

The context that is passed to the `accumulator-rule`.

#### Attribute `name`

The name of the variable that becomes available to a burst-mode rule.


#### Attribute `select`

The select that is passed to the `accumulator-rule`. `select` is optional and defaults to the current node (".").

#### Implementation

`sch:reference` tags are converted to accumulators.

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

An `xsl:variable` is introduced to the grounded portion of each generated burst-mode `sch:rule`.

```xslt
<xsl:variable name="refData" select="accumulator-after('vRefData') />
```

> The `xsl:variable` cannot be declared globally because the state of the accumulator may change as more of the streamed document is read.

Optionally, we could add a attribute to `sch:rule` like `burst-use-reference` which would allow users to control which variables get defined, somewhat akin to `use-accumulators` (XSLT3).

**Alternative Approach (2)**

Create a function in the Schematron implementation, such as `sch:ref()`, which would serve as a wrapper around `accumulator-after`.

**Alternative Approach (3)**

Extend `sch:let` instead of introducing `sch:reference`.

## Regular Accumulators

let someone else fill this out :-)

## Axis support 

This table describes some of the limitations of streaming, depending on what value you set for `xsl:rule`'s `streaming` attribute.

| rule streaming      | XSLT streaming | rule context                                                              | assert test                  |
|----------|----------|---------------------------------------------------------------------------|------------------------------|
| off      | off      | all axis                                                                  | all axis                     |
| on       | on       | current node and ancestors                                                | current node and ancestors   |
| copy-of  | on       | current node and ancestors                                                | child                        |
| snapshot | on       | current node and ancestors                                                | child and ancestor           |
| inherit  | off      | all axis (up to copied or snapshotted node), then ancestor if snapshotted | all axis (up to copied or snapshotted node), then ancestor if snapshotted |
