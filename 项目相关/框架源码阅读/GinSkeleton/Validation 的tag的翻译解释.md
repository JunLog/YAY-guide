# 关于Validation 的tag的翻译解释

validator是一个后端的简易的参数校验的库

### Fields:

| Tag        | Description                                                  |
| ---------- | ------------------------------------------------------------ |
| eqcsfield  | Field Equals Another Field (relative) 字段等于另一个字段（相对） |
| eqfield    | Field Equals Another Field 字段等于另一个字段                |
| gtcsfield  | Field Greater Than Another Relative Field  字段大于另一个相对字段 |
| gtecsfield | Field Greater Than or Equal To Another Relative Field  大于或等于另一个相对字段的字段 |
| gtefield   | Field Greater Than or Equal To Another Field 大于或等于另一个字段 |
| gtfield    | Field Greater Than Another Field 字段大于另一个字段          |
| ltcsfield  | Less Than Another Relative Field 小于另一个相对字段          |
| ltecsfield | Less Than or Equal To Another Relative Field  小于或等于另一个相对字段 |
| ltefield   | Less Than or Equal To Another Field 小于或等于另一个字段     |
| ltfield    | Less Than Another Field 小于另一个字段                       |
| necsfield  | Field Does Not Equal Another Field (relative) 字段不等于另一个字段（相对） |
| nefield    | Field Does Not Equal Another Field 字段不等于另一个字段      |

### Strings:

| Tag             | Description                                                  |
| --------------- | ------------------------------------------------------------ |
| alpha           | Alpha Only   验证字符串值仅包含 ASCII 字母字符               |
| alphanum        | Alphanumeric 将验证字符串值仅包含 ASCII 字母数字字符         |
| alphanumunicode | Alphanumeric Unicode 验证字符串值仅包含 unicode 字母字符     |
| alphaunicode    | Alpha Unicode 验证字符串值仅包含 unicode 字母数字字符        |
| ascii           | ASCII                                                        |
| boolean         | Boolean  验证了一个字符串值可以成功地被解析为一个带有 strconv.ParseBool 的布尔值 |
| contains        | Contains   验证字符串值是否包含子字符串值。                  |
| containsany     | Contains Any 验证字符串值在子字符串值中包含任何 Unicode 代码点。 |
| containsrune    | Contains Rune 验证字符串值是否包含提供的符文值。             |
| endsnotwith     | Ends With  验证字符串值是否以提供的字符串值结尾              |
| endswith        | Ends With  符串值不以提供的字符串值结尾                      |
| excludes        | Excludes  验证字符串值不包含子字符串值。                     |
| excludesall     | Excludes All  验证字符串值在子字符串值中不包含任何 Unicode 代码点。 |
| excludesrune    | Excludes Rune  验证字符串值不包含提供的符文值。              |
| lowercase       | Lowercase  验证字符串值是否仅包含小写字符。空字符串不是有效的小写字符串。 |
| multibyte       | Multi-Byte Characters  验证字符串值是否包含一个或多个多字节字符。注意：如果字符串为空，则验证为真。 |
| number          | NOT DOCUMENTED IN doc.go                                     |
| numeric         | Numeric  验证字符串值是否包含基本数值。基本不包括指数等......对于整数或浮点数，它返回true。 |
| printascii      | Printable ASCII  验证字符串值是否仅包含可打印的 ASCII 字符。注意：如果字符串为空，则验证为真。 |
| startsnotwith   | Starts Not With                                              |
| startswith      | Starts With                                                  |
| uppercase       | Uppercase                                                    |

### Format:

| Tag                           | Description                                                  |
| ----------------------------- | ------------------------------------------------------------ |
| base64                        | Base64 String 验证字符串值是否包含有效的 base64 值。尽管空字符串是有效的 base64，但这会将空字符串报告为错误，如果您希望接受空字符串为有效，您可以将其与 omitempty 标签一起使用。 |
| base64url                     | Base64URL String                                             |
| bic                           | Business Identifier Code (ISO 9362)                          |
| bcp47_language_tag            | Language tag (BCP 47)                                        |
| btc_addr                      | Bitcoin Address                                              |
| btc_addr_bech32               | Bitcoin Bech32 Address (segwit)                              |
| datetime                      | Datetime                                                     |
| e164                          | e164 formatted phone number                                  |
| email                         | E-mail String                                                |
| eth_addr                      | Ethereum Address                                             |
| hexadecimal                   | Hexadecimal String                                           |
| hexcolor                      | Hexcolor String                                              |
| hsl                           | HSL String                                                   |
| hsla                          | HSLA String                                                  |
| html                          | HTML Tags                                                    |
| html_encoded                  | HTML Encoded                                                 |
| isbn                          | International Standard Book Number                           |
| isbn10                        | International Standard Book Number 10                        |
| isbn13                        | International Standard Book Number 13                        |
| iso3166_1_alpha2              | Two-letter country code (ISO 3166-1 alpha-2)                 |
| iso3166_1_alpha3              | Three-letter country code (ISO 3166-1 alpha-3)               |
| iso3166_1_alpha_numeric       | Numeric country code (ISO 3166-1 numeric)                    |
| iso3166_2                     | Country subdivision code (ISO 3166-2)                        |
| iso4217                       | Currency code (ISO 4217)                                     |
| json                          | JSON                                                         |
| jwt                           | JSON Web Token (JWT)                                         |
| latitude                      | Latitude                                                     |
| longitude                     | Longitude                                                    |
| postcode_iso3166_alpha2       | Postcode                                                     |
| postcode_iso3166_alpha2_field | Postcode                                                     |
| rgb                           | RGB String                                                   |
| rgba                          | RGBA String                                                  |
| ssn                           | Social Security Number SSN                                   |
| timezone                      | Timezone                                                     |
| uuid                          | Universally Unique Identifier UUID                           |
| uuid3                         | Universally Unique Identifier UUID v3                        |
| uuid3_rfc4122                 | Universally Unique Identifier UUID v3 RFC4122                |
| uuid4                         | Universally Unique Identifier UUID v4                        |
| uuid4_rfc4122                 | Universally Unique Identifier UUID v4 RFC4122                |
| uuid5                         | Universally Unique Identifier UUID v5                        |
| uuid5_rfc4122                 | Universally Unique Identifier UUID v5 RFC4122                |
| uuid_rfc4122                  | Universally Unique Identifier UUID RFC4122                   |

### Comparisons:

| Tag  | Description                                     |
| ---- | ----------------------------------------------- |
| eq   | Equals   ne 将确保该值  等于给定的参数。        |
| gt   | Greater than  这将确保该值大于给定的参数。      |
| gte  | Greater than or equal  这将确保该值大于或者等于 |
| lt   | Less Than 小于                                  |
| lte  | Less Than or Equal   小于等于                   |
| ne   | Not Equal  不等于的                             |

### Other:

| Tag                  | Description                                                  |
| -------------------- | ------------------------------------------------------------ |
| dir                  | Directory                                                    |
| file                 | File path                                                    |
| isdefault            | Is Default                                                   |
| len                  | Length                                                       |
| max                  | Maximum 对于数字，max将确保该值小于或等于给定的参数。对于字符串，它检查字符串长度是否最多为该数量的字符。对于切片、数组和映射，验证项目数。 |
| min                  | Minimum 对于数字，min将确保该值大于或等于给定的参数。对于字符串，它检查字符串长度是否至少为该数量的字符。对于切片、数组和映射，验证项目数。 |
| oneof                | One Of  对于字符串、int和uint，其中一个将确保该值是参数中的值之一。参数应该是由空格分隔的值列表。值可以是字符串或数字。要将字符串与其中的空格匹配，请在单引号之间包含目标字符串。 |
| required             | Required                                                     |
| required_if          | Required If                                                  |
| required_unless      | Required Unless                                              |
| required_with        | Required With                                                |
| required_with_all    | Required With All                                            |
| required_without     | Required Without                                             |
| required_without_all | Required Without All                                         |
| excluded_with        | Excluded With                                                |
| excluded_with_all    | Excluded With All                                            |
| excluded_without     | Excluded Without                                             |
| excluded_without_all | Excluded Without All                                         |
| unique               | Unique                                                       |

#### Aliases:

| Tag          | Description                                                 |
| ------------ | ----------------------------------------------------------- |
| iscolor      | hexcolor\|rgb\|rgba\|hsl\|hsla                              |
| country_code | iso3166_1_alpha2\|iso3166_1_alpha3\|iso3166_1_alpha_numeric |