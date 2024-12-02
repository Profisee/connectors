# Table of Contents

- [Table of Contents](#table-of-contents)
- [REST Service Provider Artifact](#rest-service-provider-artifact)
  - [REST Specification Record](#rest-specification-record)
  - [Service Provider Record](#service-provider-record)
    - [Service Provider Configuration](#service-provider-configuration)
- [Document Extension](#document-extension)
  - [Path](#path)
    - [Operation](#operation)
  - [Components Extension](#components-extension)
    - [Parameter (Components)](#parameter-components)
      - [Parameter Assignment](#parameter-assignment)
      - [Presenter Configuration](#presenter-configuration)
        - [Picklist Settings](#picklist-settings)
        - [DateTime Settings](#datetime-settings)
        - [Number Settings](#number-settings)
        - [ParameterSet Settings](#parameterset-settings)
        - [String Settings](#string-settings)
      - [Formatter Configuration](#formatter-configuration)
    - [DataSource (Components)](#datasource-components)
    - [Schema (Components)](#schema-components)
    - [Security Scheme (Components)](#security-scheme-components)
      - [OAuth2 Flows](#oauth2-flows)
        - [OAuth2 Client Credentials Flow](#oauth2-client-credentials-flow)
- [Enums](#enums)
  - [DataSourceType](#datasourcetype)
  - [ConfigurationScope](#configurationscope)
  - [OAuth2 Client Credential Style](#oauth2-client-credential-style)
  - [OAuth2 Header Style](#oauth2-header-style)
  - [ParameterDataType](#parameterdatatype)
  - [ParameterFormatterType](#parameterformattertype)
  - [ParameterAssignmentType](#parameterassignmenttype)
  - [OperationType](#operationtype)
  - [SecuritySchemeType](#securityschemetype)
  - [DateTimePickerControls](#datetimepickercontrols)

# REST Service Provider Artifact

The REST service provider artifact contains all the configuration required for Connect to integrate with external REST APIs. A new service provider should be created for each service and can define multiple endpoints with shared schemas and UI configuration.

The artifact can be imported and exported from the Connect admin area in the Profisee Portal.

| Name       | Type       | Description          |
|------------|------------|----------------------|
| $type | `string` | Static value, must be set to: `Profisee.Platform.ConnEx.Contracts.Administration.RestServiceProviderArtifact, Profisee.Platform.ConnEx` |
| ArtifactVersion | `string` | Static value, must be set to: `1.0.1.0` |
| RestSpecificationRecord | [RestSpecification](#rest-specification-record) | The REST specification record with the OpenAPI Document. |
| ServiceProviderRecord | [ServiceProviderRecord](#service-provider-record) | The service provider configuration record containing the Connect OpenAPI extensions. |

## REST Specification Record

The REST specification record contains the OpenAPI document for the service Connect will integrate with.

The OpenAPI document specification is a standard for describing an API and it's various endpoints and request/response schemas. The specification can be found at the [Swagger OpenAPI Specification](https://swagger.io/specification/).

Many services provide the document for their own APIs or they can be found in various directories such as [APIs.guru OpenAPI Directory](https://github.com/APIs-guru/openapi-directory)

The `OpenApiDocument` field contains the entire document as string escaped JSON. Connect supports OpenAPI Specification v3.0.x.

| Name       | Type       | Description          |
|------------|------------|----------------------|
| $type | `string` | Static value, must be set to: `Profisee.Platform.ConnEx.Contracts.Administration.RestSpecification, Profisee.Platform.ConnEx` |
| Name | `string` | The name of the REST Specification Record. This will not be displayed. |
| Code | `string` | A Guid id for the record. |
| OpenApiDocument | `string` | The string escaped OpenAPI document. |

## Service Provider Record

The service provider record represents the provider in the Profisee Portal and contains the configuration object.

The `Name` will be used throughout the administration experience to reference this provider.

| Name       | Type       | Description          |
|------------|------------|----------------------|
| $type | `string` | Static value, must be set to: `Profisee.Platform.ConnEx.Contracts.Administration.ServiceProviderConfigurationRecord, Profisee.Platform.ConnEx` |
| Id | `string` | A Guid id for the record. |
| Name | `string` | A name for the Service Provider. |
| Type | `string` | Static value, must be set to: `Rest` |
| Configuration | [ServiceProviderConfiguration](#service-provider-configuration) | The service provider configuration. |

### Service Provider Configuration

The service provider configuration object contains all of the settings for the service provider.

The `RestSpecificationId` must match the `Code` field of the [RESTSpecification](#rest-specification-record) which links the configuration to the OpenAPI document.

| Name       | Type       | Description          |
|------------|------------|----------------------|
| $type | `string` | Static value, must be set to: `Profisee.Platform.ConnEx.Contracts.Administration.RestServiceProviderConfiguration, Profisee.Platform.ConnEx` |
| RestSpecificationId | `string` | The Code of the REST specification. |
| OpenApiExtension | [OpenApiDocumentExtension](#document-extension) | The Connect OpenAPI document extension. |

---

# Document Extension

The document extension is used to extend the OpenAPI document provided by a service in order to customize it's behavior and functionality in Connect and the administration in the Profisee Portal. This is the main area that will need to be built out when creating a Connect integration with a new service.

All endpoints which Connect will use from the service need to be defined in `Paths`. The key will be the relative path to the individual endpoint from the OpenAPI document's [Paths](https://swagger.io/docs/specification/paths-and-operations/). For example `/pets/{petId}`.

The `Security` list should contain the names of any security schemes to enable for the service provider. Adding the security scheme here will enable it for all paths and operations. They can also be enabled at a single operation level in the [operations extension](#operation). The names come from the OpenAPI document's [Security Schemes](https://swagger.io/docs/specification/authentication/).

| Name       | Type       | Description          |
|------------|------------|----------------------|
| Paths      | Dictionary<string, [Path](#path)> | A dictionary of path extension objects. |
| Components | [OpenApiComponentsExtension](#components-extension) | The extensions for the OpenAPI components. |
| Security   | List\<string> | A list of security schemes Connect will use. |

---

## Path

The `FriendlyName` is used in the Profisee Portal admin experience to reference the path.

| Name       | Type       | Description          |
|------------|------------|----------------------|
| FriendlyName | `string` | A friendly name for the path to reference it in the Profisee Portal. |
| Operations | Dictionary<[OperationType](#operationtype), [Operation](#operation)> | A dictionary of OpenApiOperationExtension objects. |

### Operation

`Parameters` is a dictionary mapping an operation's path or query string parameters to our extension parameter name. The key will come from the OpenAPI document [operation parameters](https://swagger.io/docs/specification/paths-and-operations/) and the value will be the key in our extension [parameters](#parameter-components).

`ResponseMappingSchemas` optionally allows defining which property of the response will be used as the root for response mapping back to Profisee. The key of the dictionary will be the HTTP status code of the response, for example `200` or `500`. If a schema is not defined for the status code, it will default back to the root schema of the response.

For example if the endpoint's response contains an array of records that you want to use for response mapping you would do the following:

Response:

``` json
{
  "Records": [
    { "Id": "1", "Name": "ABC" }
  ]
}
```

ResponseMappingSchemas:

``` json
{
  "ResponseMappingSchemas": {
    "200": "Records"
  }
}
```

The `Security` list should contain the names of any security schemes to enable for the this specific operation. The names come from the OpenAPI document's [Security Schemes](https://swagger.io/docs/specification/authentication/).

| Name       | Type       | Description          |
|------------|------------|----------------------|
| BaseUrl    | `string`   | The base URL for the operation. |
| FriendlyName | `string` | A friendly name for the operation to reference it in the Profisee Portal. |
| Parameters | Dictionary<string, string> | A dictionary of parameters. |
| ResponseMappingSchemas | Dictionary<string, string> | A dictionary of response mapping schemas. |
| Security   | List\<string> | A list of security schemes. |
| MaximumRecordsPerRequest | `int` | The maximum number of records per request that the operation supports. |

---

## Components Extension

The components extension is used to extend the OpenAPI document's [components section](https://swagger.io/docs/specification/components/). This is where we define the custom parameters, data sources, schemas, and security schemes that Connect will use.

Keys for `Parameters` will be parameter names that will be used in the [path extension](#path) and [schemas extension](#schema-components) to reference the parameter.

Keys for `DataSources` will be custom data source names that will be used in the [presenter configuration](#presenter-configuration) to reference the data source.

Keys for `Schemas` should match the OpenAPI document's [schemas](https://swagger.io/docs/specification/data-models/).

Keys for `SecuritySchemes` should match the OpenAPI document's [Security Schemes](https://swagger.io/docs/specification/authentication/).

| Name       | Type       | Description          |
|------------|------------|----------------------|
| Parameters | Dictionary<string, [Parameter](#parameter-components)> | A dictionary of ParameterConfiguration objects. |
| DataSources | Dictionary<string, [DataSource](#datasource-components)> | A dictionary of DataSourceConfiguration objects. |
| Schemas    | Dictionary<string, [Schema](#schema-components)> | A dictionary of OpenApiSchemaExtension objects. |
| SecuritySchemes | Dictionary<string, [SecurityScheme](#security-scheme-components)> | A dictionary of OpenApiSecuritySchemeExtension objects. |

---

### Parameter (Components)

Parameters are used to define which fields will be available in the Profisee Portal for configuring the request to a service. They can be used to define static values, expressions, or credentials that will be sent with the request.

| Name       | Type       | Description          |
|------------|------------|----------------------|
| Scope      | [ConfigurationScope](#configurationscope) | The scope of the parameter. |
| FriendlyName | `string` | A friendly name for the parameter that will be shown in the Profisee Portal. |
| DefaultValue | [ParameterAssignment](#parameter-assignment) | An optional default value for the parameter. |
| PresenterConfiguration | [PresenterConfiguration](#presenter-configuration) | The presenter configuration for the parameter. |
| FormatterConfiguration | [FormatterConfiguration](#formatter-configuration) | An optional formatter configuration for the parameter. |

#### Parameter Assignment

A parameter assignment can be provided as a default value for a parameter.

Literal value example:

``` json
{
  "AssignmentType": "Literal",
  "Value": "123",
  "DataType": "String"
}
```

Expression value example:

``` json
{
  "AssignmentType": "Expression",
  "Value": {
    "ExpressionText": "NEWGUID"
  },
  "DataType": "Object"
}
```

| Name       | Type       | Description          |
|------------|------------|----------------------|
| AssignmentType | [ParameterAssignmentType](#parameterassignmenttype) | The type of parameter assignment. |
| FormatterConfiguration | [FormatterConfiguration](#formatter-configuration) | An optional formatter configuration for the parameter. |
| Value      | [JToken](https://www.newtonsoft.com/json/help/html/T_Newtonsoft_Json_Linq_JToken.htm) | The value of the parameter. |
| DataType   | [JTokenType](https://www.newtonsoft.com/json/help/html/T_Newtonsoft_Json_Linq_JTokenType.htm) | The data type of the parameter value. |

#### Presenter Configuration

Presenter configuration is used to define how the parameter will be exposed for editing in the Profisee Portal. The `Type` will determine the type of UI control used to present the parameter. The `Settings` object will contain any additional configuration for the presenter based on the type.

| Name       | Type       | Description          |
|------------|------------|----------------------|
| Type       | [ParameterDataType](#parameterdatatype) | The type of parameter data. |
| Settings   | [JObject](https://www.newtonsoft.com/json/help/html/T_Newtonsoft_Json_Linq_JObject.htm) | The settings for the parameter presenter. |

##### Picklist Settings

Picklist settings are used for the `Picklist` parameter data type. The `DataSource` should be the name of a data source defined in the components [datasource extension](#datasource-components).

| Name            | Type    | Description |
|-----------------|---------|-------------|
| DataSource      | string  | The data source to be used. |
| AllowMultiselect| bool    | Indicates if multiple selections are allowed. Default is `false`. |
| RequireSelection| bool    | Indicates if selection is required. Default is `false`. |

##### DateTime Settings

The date time settings are used for the `DateTime` parameter data type. The `Controls` should be set to the type of date time picker to use.

| Name     | Type                      | Description |
|----------|---------------------------|-------------|
| Controls | [DateTimePickerControls](#datetimepickercontrols)    | Specifies the controls for the DateTimePicker. Default is `DateAndTime`. |

##### Number Settings

The number settings are used for the `Number` parameter data type. It can be used to define optional minimum and maximum values and the number of decimal places.

| Name          | Type    | Description |
|---------------|---------|-------------|
| DecimalPlaces | uint    | The maximum number of digits to the right of the decimal point. Default is `0`. |
| MinimumValue  | double | An optional minimum value allowed. |
| MaximumValue  | double | An optional maximum value allowed. |

##### ParameterSet Settings

A `ParameterSet` is a collection of parameters that should be sent as an object in a request. The `Parameters` should be a list of parameter names defined in the components [parameters extension](#parameter-components).

| Name       | Type                           | Description |
|------------|--------------------------------|-------------|
| Parameters | Dictionary<string, string>     | A dictionary of parameters. |

##### String Settings

The string settings are used for the `String` parameter data type. It can be used to define optional minimum and maximum lengths.

| Name          | Type    | Description |
|---------------|---------|-------------|
| MinimumLength | int    | An optional minimum number of characters allowed. |
| MaximumLength | int    | An optional maximum number of characters allowed. If set to a value greater than `4000`, it will be capped at `4000`. |

#### Formatter Configuration

Formatter configuration can be provided to define how arrays or object are formatted when sent in a request. The `Type` will determine the type of formatter used.

OpenApiParameterFormatter:

- Arrays will be formatted as a comma separated list of values.
- Objects will be formatted as `key=value` pairs separated by `&`.

ExtendedOpenApiParameterFormatter:

- Arrays will be formatted as `arrayname=value` pairs separated by `&`.
- Objects will be formatted as `key:value` pairs separated by `;`.

| Name       | Type       | Description          |
|------------|------------|----------------------|
| Type       | [ParameterFormatterType](#parameterformattertype) | The type of parameter formatter. |

---

### DataSource (Components)

A data source is a reference to an OpenAPI document [enum schema](https://swagger.io/docs/specification/data-models/enums/) which will pull values to be used by dropdowns or other UI controls in the Profisee Portal. Once defined here, the data source can be used in the [presenter configuration](#presenter-configuration) settings of a parameter.

Example schema from an OpenAPI document:

``` json
{
  "components": {
    "schemas": {
      "OffOnDomain": {
        "type": "string",
        "enum": [
          "Off",
          "On"
        ]
      }
    }
  }
}
```

Data source configuration:

``` json
{
  "Type": "OpenApi",
  "Locator": "#/components/schemas/OffOnDomain"
}
```

| Name       | Type       | Description          |
|------------|------------|----------------------|
| Type       | [DataSourceType](#datasourcetype) | The type of data source. |
| Locator    | `string`   | The locator for the data source. |

---

### Schema (Components)

Schemas should contain all of the property names that Connect will use from the request schema of the OpenAPI document [operation request body](https://swagger.io/docs/specification/describing-request-body/).

| Name       | Type       | Description          |
|------------|------------|----------------------|
| Parameters | List\<string> | A list of parameters. |

---

### Security Scheme (Components)

Connect supports the OAuth2 Client Credentials flow which must be defined in the OpenAPI document's [Security Schemes](https://swagger.io/docs/specification/authentication/oauth2/).

| Name       | Type       | Description          |
|------------|------------|----------------------|
| Type       | [SecuritySchemeType](#securityschemetype) | The type of security scheme. |
| Flows      | [OAuth2Flows](#oauth2-flows) | The OAuth2 flows object. |

#### OAuth2 Flows

| Name       | Type       | Description          |
|------------|------------|----------------------|
| ClientCredentials | [OAuth2ClientCredentialsFlow](#oauth2-client-credentials-flow) | The client credentials flow object. |

##### OAuth2 Client Credentials Flow

`BasicAuthenticationHeaderStyle` is only used if `ClientCredentialStyle` is set to AuthorizationHeader. The header style will default to `Rfc6749` is none is provided. `Rfc2617` is available for backwards compatibiliy to support some services which still use it.

| Name       | Type       | Description          |
|------------|------------|----------------------|
| ClientCredentialStyle | [OAuth2ClientCredentialStyle](#oauth2-client-credential-style) | The client credential style. |
| BasicAuthenticationHeaderStyle | [OAuth2HeaderStyle](#oauth2-header-style) | The basic authentication header style. |

# Enums

## DataSourceType

| Name       | Description          |
|------------|----------------------|
| OpenApi    | OpenAPI data source type which pulls values from an OpenAPI document schema. |

## ConfigurationScope

| Name       | Description          |
|------------|----------------------|
| Service    | Parameters defined in the Service scope will be configurable in the Service Configuration and will be sent with all requests to a specific operation. |
| Data       | Parameters defined in the Data scope will be available in the Integration Strategy request mappings and be evaluated per record. Use this to access data from records. |

## OAuth2 Client Credential Style

| Name               | Description          |
|--------------------|----------------------|
| AuthorizationHeader| Authorization header style. |
| PostBody           | Post body style. |

## OAuth2 Header Style

| Name       | Description          |
|------------|----------------------|
| Rfc6749    | Uses the encoding as described in the OAuth 2.0 spec. |
| Rfc2617    | Uses the encoding as described in the original basic authentication spec. |

## ParameterDataType

| Name           | Description                |
|----------------|----------------------------|
| String         | String data type.          |
| Number         | Number data type.          |
| DateTime       | DateTime data type.        |
| Picklist       | Picklist data type.        |
| Credential     | Credential data type.      |
| Expression     | Expression data type.      |
| ParameterSet   | Parameter set data type.   |

## ParameterFormatterType

| Name                          | Description                        |
|-------------------------------|------------------------------------|
| OpenApiParameterFormatter     | OpenAPI parameter formatter.       |
| ExtendedOpenApiParameterFormatter | Extended OpenAPI parameter formatter. |

## ParameterAssignmentType

| Name            | Description                        |
|-----------------|------------------------------------|
| Literal         | Literal value assignment.          |
| Expression      | Expression-based assignment.       |

## OperationType

| Name    | Description                                                                 |
|---------|-----------------------------------------------------------------------------|
| Get     | Represents an HTTP GET operation, typically used to retrieve data from the server. |
| Put     | Represents an HTTP PUT operation, typically used to update or replace existing data on the server. |
| Post    | Represents an HTTP POST operation, typically used to create new data on the server. |
| Delete  | Represents an HTTP DELETE operation, typically used to delete existing data from the server. |
| Patch   | Represents an HTTP PATCH operation, typically used to apply partial modifications to a resource. |

## SecuritySchemeType

| Name    | Description                                                                 |
|---------|-----------------------------------------------------------------------------|
| OAuth2  | OAuth2 security scheme. |

## DateTimePickerControls

| Name         | Description |
|--------------|-------------|
| Date         | Provides a date picker experience for selecting a date in addition to input fields for entering the day, month, and year. |
| DateAndTime  | Provides a date picker experience for selecting a date in addition to input fields for entering the day, month, year, hour, and minute. |
