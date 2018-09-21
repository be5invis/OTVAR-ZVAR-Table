# OpenType Extension Draft: `ZVAR` table

**Renzhi Li @ Microsoft, January 2018**

This document describes a proposed new Axes Variations (`ZVAR`) table that provides variation data to allow non-linear variation across a font’s variation space with inter-axis effects, and hiding parametric axes behind the optical axes, and provide language-dependent and layout-dependent axes mappings for multiple-language fonts.

## Background

### Optical versus Parametric

Type Network has purposed a set of parametric axes, however how should they interact with existing optical axes, is still unknown. In their current implementation, their example font is provided with both parametric and optical axes provided, however manipulating them simultaneously would result a broken result.

### Layout-dependent axes remapping 

The current `fvar` definition could have problems when dealing with fonts covering multiple, unrelated scripts, like a typical Japanese font, which convers Latin glyphs and Ideographic glyphs. When a Japanese font is used in vertical layout, the Latin characters should be rotated by 90° clockwise, while ideographs are kept upright. Therefore, if the font provides width variation (`wdth` axes), then the following situation would took place:

![WDTH problem](https://user-images.githubusercontent.com/240091/34754331-1702aaaa-f5f8-11e7-86dd-f14ac75a606f.png)

The `wdth` axis affects the Latins on their *height* (since they are rotated), and ideographs on their *width*. Such variation would lead to inconsistency when the text is layout vertically.

Therefore we need some mechanism that "swaps" the width and height axis for a certain scope of glyphs (like ideographs, if the "width" is defined by the dimension on the line direction) only in vertical writing direction. 

### Non-linear axes mapping

-- TBD --

## Overview

The main idea of `ZVAR` table is "another layer of indirection". A `ZVAR` table links the User Axes defined in `STAT` table, to `fvar` design axes set, using various functions called Mappers. In a `ZVAR` table there could be multiple Mappers, and every one of them are limited to one specific layout mode.

A `ZVAR` table **should always** co-exist with a `STAT` table, and **should never** co-exist with `avar` tables. A proper implementation should handle these cases.

## STAT table modifications

*Style attributes header:*

| Type     | Name                       | Description                                                  |
| -------- | -------------------------- | ------------------------------------------------------------ |
| UINT16   | `majorVersion`             | Major version number of the style attributes table — set to 1. |
| uint16   | `minorVersion`             | Minor version number of the style attributes table — set to **3**. |
| uint16   | `designAxisSize`           | The size in bytes of each axis record.                       |
| uint16   | `designAxisCount`          | The number of design axis records. In a font with an `fvar` table **and without a `ZVAR` table**, this value must be greater than or equal to the `axisCount` value in the `fvar` table. In all fonts, must be greater than zero if `axisValueCount` is greater than zero. |
| Offset32 | `designAxesOffset`         | Offset in bytes from the beginning of the `STAT` table to the start of the design axes array. If `designAxisCount` is zero, set to zero; if `designAxisCount` is greater than zero, must be greater than zero. |
| uint16   | `axisValueCount`           | The number of axis value tables.                             |
| Offset32 | `offsetToAxisValueOffsets` | Offset in bytes from the beginning of the `STAT` table to the start of the design axes value offsets array. If `axisValueCount` is zero, set to zero; if `axisValueCount` is greater than zero, must be greater than zero. |
| uint16   | `elidedFallbackNameID`     | Name ID used as fallback when projection of names into a particular font model produces a subfamily name containing only elidable elements. |

*AxisRecord:*

| Type       | Name         | Description                                                  |
| ---------- | ------------ | ------------------------------------------------------------ |
| Tag        | axisTag      | A tag identifying the axis of design variation.              |
| UINT16     | axisNameID   | The name ID for entries in the 'name' table that provide a display string for this axis. |
| UINT16     | axisOrdering | A value that applications can use to determine primary sorting of face names, or for ordering of descriptors when composing family or face names. |
| ------     | ------------ | *The following fields must be present if `ZVAR` table presents.* |
| **UINT16** | **`flags`**  | **Axis qualifiers — see `fvar`.**                            |

## Table Format

### Terminology

- **User Axis** (*pl.* User Axes): An axis exposed to users and `STAT` table. The tag, name, range of User Axes is defined in `STAT` table.

- **Design Axis** (*pl.* Design Axes): The existing axes set defined in `fvar` table. In a font containing a `ZVAR` table, all the design axes should be marked as hidden.

- **Mapper**: A mathematical function defined in `ZVAR` table to map User Axes values to Design Axes values. It is defined by multiple Formulae, and each Formula is used to produce the value for one Design Axis.

- **Formula**: A mathematical function defined in `ZVAR` table to map User Axes values to one Fixed number, which would be used by one Design Axis.

  The output of all forumlae are **not normalized**. They should be further normalized using the min/default/max values defined in `fvar` table.

### Root Structure — `ZVAR` table

The `ZVAR` table as the following format.

#### ZVAR Table (`ZVARTable`)

| Type                 | Name                           | Description                                                  |
| -------------------- | ------------------------------ | ------------------------------------------------------------ |
| UINT16               | `majorVersion`                 | Table major version - set to `1`.                            |
| UINT16               | `minorVersion`                 | Table minor version - set to `0`.                            |
| UINT16               | `numberOfFvarAxes`             | Number of design axes. Must equal to ` designAxisCount ` in `fvar` table. |
| UINT16               | `numberOfUserAxes`             | Number of user axes. Must equal to ` designAxisCount ` in `STAT` table. |
| UINT16               | `numberOfMappers`              | Number of mappers. Must greater than zero.                   |
| Offset32\<`Mapper`\> | `mappers`\[`numberOfMappers`\] | The mapper array.                                            |

### Mappers

A mapper defines a function that maps the value array of user axes (typed as `Fixed[numberOfUserAxes]`) to the value array of design axes (typed as `Fixed[numberOfFvarAxes]`), used in a specific layout mode (like horizontal / vertical layout).

A mapper contains an array of *formulae*, each formula maps the **non-normalized** user axes value array to one `Fixed` value, representing the **non-normalized** value of each design axis. The format of each formula is specified by a `formulaType` field.

#### Mapper Record (`Mapper`)

**V2 Notes**: The `scriptTag` and `languageTag` is unnecessary since we could use `locl` features to being axes re-mapping by using different glyphs. Therefore these fields are removed. `layoutModeTag` is kept to handle layout-dependent axes remapping.

| Type                | Name                             | Description                                         |
| ------------------- | -------------------------------- | --------------------------------------------------- |
| Tag                 | `layoutModeTag`                  | Currently only `DFLT`, `hori` and `vert` are valid. |
| Offset32<`Formula`> | `formulae`\[`numberOfFvarAxes`\] | List of mapping formulae.                           |

```cpp
int scoreMapper (const Tag layout, const mapper* mapper) {
    int result = 0;
    if (mapper->layoutModeTag == layout) {
        result += 2;
    } else if(mapper->layoutModeTag == 'DFLT') {
        result += 1;
    }
    return result;
}

Fixed* evalZvar (const ZVARTable* zvar, const FVARTable* fvar, const Fixed* vals,
                 const Tag layout) {
    Fixed *results = new Fixed[zvar->numberOfFvarAxes];
    for (UINT16 j = 0; j < fvar->axisCount; j++) {
        results[j] = fvar->axes[j].defaultValue;
    }
    int optimalMapperScore = 0;
    Mapper* optimalMapper = &zvar->mappers[0];
    for (UINT16 j = 0; j < zvar->numberOfMappers; j++) {
        int mapperScore = scoreMapper(layout, &zvar->mappers[j]);
        if(mapperScore > optimalMapperScore) {
            optimalMapperScore = mapperScore;
            optimalMapper = &zvar->mappers[j];
        }
    }
    if (optimalMapperScore > 0) {
        for (UINT16 j = 0; j < fvar->axisCount; j++) {
            results[j] = evalFormula(vals, &optimalMapper->formulae[j]);
        }
    }
    return results;
}
```

#### Formula (`Formula`)

| Type                       | Name          | Description               |
| -------------------------- | ------------- | ------------------------- |
| UINT8                      | `formulaType` | Type of this forumla.     |
| `Formula`\<`formulaType`\> | `formulaBody` | Contents of this formula. |

Evaluating one formula providing a user is value list is defined as:
```cpp
Fixed evalFormula (const Fixed* vals, const Formula* form) {
    return evalFormula[form->formulaType](vals, &form->body);
}
```

#### Formula type 0 (Copy mapping, `Formula<0>`)

This type of formula copys a **non-normalized** user axis value to a design axis, as its **non-normalized** value.

| Type   | Name        | Description         |
| ------ | ----------- | ------------------- |
| UINT16 | `axisIndex` | Index of user axis. |

Evaluating one formula providing a user is value list is defined as:
```cpp
Fixed evalFormula[0](const Fixed* vals, const Formula<0>* form) {
    return result[form->axisIndex];
}
```

#### Formula type 1 (`Formula<1>`)

Formula Type 1 contains a *constant term* and multiple *non-constant terms*. Each non-constant term is a product of tent-shaped functions, applied to various User Axes. The values of user axes are also **non-normalized**.

| Type           | Name                   | Description                   |
| -------------- | ---------------------- | ----------------------------- |
| Fixed          | `constantTerm`         | Constant term of the formula. |
| UINT16         | `termCount`            | Number of non-constant terms. |
| `Formula1Term` | `terms`\[`termCount`\] | Non-constant terms.           |

Evaluating one formula providing a user is value list is defined as:
```cpp
Fixed evalFormula[1](const Fixed* vals, const Formula<1>* form) {
    Fixed result = form->constantTerm;
    for (UINT16 j = 0; j < form->termCount; j++) {
        result += weightTerm(vals, &form->terms[j]);
    }
    return result;
}
```

#### Format type 1 -- Term (`Formula1Term`)

| Type             | Name                       | Description          |
| ---------------- | -------------------------- | -------------------- |
| UINT16           | `factorCount`              | Quantity of factors. |
| `Formula1Factor` | `factors`\[`factorCount`\] | Factors              |

Evaluating one term providing a user axis value list is defined as:

```cpp
Fixed evalTerm (const Fixed* vals, const Formula1Term* term) {
    Fixed result = 1;
    for (UINT16 j = 0; j < term->factorCount; j++) {
        const Formula1Factor *factor = &term->factors[j]
        result *= weightFactor(vals[factor->axisIndex], factor);
    }
    return result;
}
```

#### Format type 1 -- Factor (`Formula1Factor`)

| Type   | Name          | Description                          |
| ------ | ------------- | ------------------------------------ |
| UINT16 | `axisIndex`   | Tag of the axis for this factor.     |
| Fixed  | `coefficient` | Coefficient.                         |
| Fixed  | `minValue`    | Min parameter of the Tent function.  |
| Fixed  | `peakValue`   | Peak parameter of the Tent function. |
| Fixed  | `maxValue`    | Max parameter of the Tent function.  |

The factor function is defined as:

```cpp
Fixed weightFactor (const Fixed val, const Formula1Factor* factor) {
    if (!factor->power) return factor->coefficient;
    const Fixed tentVal = tent(val, factor->min, factor->peak, factor->max);
    return factor->coefficient * pow(tentVal, factor.power);
}

Fixed tent (const Fixed val,
            const Fixed min, const Fixed peak, const Fixed max) {
    if (min > peak || peak > max || min >= max) { return 1; } // ignore this factor
    if (val == peak) { return 1; }                            // weight being 1
    if (val <= min || val >= max) { return 0; }
    if (val < peak) { return (val - min) / (peak - min); }
    if (val > peak) { return (max - val) / (max - peak); }
    return return 1;
}
```

