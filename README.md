# HelloID-Conn-Prov-Target-NedapOns-Deskundigheid

> [!IMPORTANT]
> This repository contains the connector and configuration code only. The implementer is responsible to acquire the connection details such as username, password, certificate, etc. You might even need to sign a contract or agreement with the supplier before implementing this connector. Please contact the client's application manager to coordinate the connector requirements.

> [!NOTE]
> This connector only associates a HelloID person with their employee accounts in Nedap Ons and updates the education (AKA Deskundigheid). Please refer to the [Fact Sheet](#fact-sheet) for more details.
>
> __Extensive knowledge of HelloID Provisioning and Nedap Ons (both Nedap user and Nedap employee) is required.__



<p align="center">
  <img src="https://user-images.githubusercontent.com/68013812/94918899-c672c700-04b3-11eb-9132-7125bbf77fa5.png"
   alt="drawing" style="width:300px;"/>
</p>

> [!IMPORTANT]
>  **Upgrade Warning**
> This V2 version of the connector has new names for the columns in the Education.csv mapping file. Make sure to change that accordingly in your Education.csv file when upgrading.


## Table of contents

- [HelloID-Conn-Prov-Target-NedapOns-Weekkaart](#helloid-conn-prov-target-nedapons-weekkaart)
  - [Table of contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Getting started](#getting-started)
    - [Prerequisites](#prerequisites)
    - [Connection settings](#connection-settings)
    - [Correlation configuration](#correlation-configuration)
    - [Available lifecycle actions](#available-lifecycle-actions)
    - [Field mapping](#field-mapping)
      - [The Primary Contract Calculation foreach employment](#the-primary-contract-calculation-foreach-employment)
      - [Script Mapping lookup values](#script-mapping-lookup-values)
      - [Cluster Configuration](#cluster-configuration)
  - [Remarks](#remarks)
    - [Correlation](#correlation)
    - [Primary Contract](#primary-contract)
    - [IO Import validation](#io-import-validation)
    - [OutputInfo](#outputinfo)
  - [Employee Additional Mapping:](#employee-additional-mapping)
  - [Supported Properties](#supported-properties)
  - [Fact Sheet](#fact-sheet)
  - [Development resources](#development-resources)
    - [API endpoints](#api-endpoints)
    - [API documentation](#api-documentation)
  - [Getting help](#getting-help)
  - [HelloID docs](#helloid-docs)

## Introduction
This repository only contains the documention. The source code can be found in a private repository and is only meant for internal use.

Nedap Ons provides an API and XML import method to programmatically interact with its services and data. This connector only updates the _registration profile_.

## Getting started

### Prerequisites

- **Nedap PFX Certificate** <br>
  A valid Nedap certificate (.PFX) *Tools4ever need to requests a certificate by Nedap to access the REST API*

- **Credentials IOImport**<br>
  Credentials for the IO Import *Different account credentials as the REST API*

- **Mapping Files**<br>
  Mapping between HR departments / functions and the Education profile (Deskundigheid).

- **Custom HelloId Property**<br>
  A custom property on the HelloID Person contract with a combination of the employeeCode and EmploymentCode called __custom.NedapOnsIdentificationNo__

### Connection settings

The following settings are required to connect to the API.

| Setting                       | Description                                                                  | Mandatory |
| ----------------------------- | ---------------------------------------------------------------------------- | --------- |
| Environment (Rest)            | The Nedap Environment of the API (Development, Staging, Production )         | Yes       |
| Certificate (.PFX) Path       | <FullPath to Certificate> Nedap-Cert.pfx                                     | Yes       |
| Certificate Password          | Password of the certificate                                                  | Yes       |
| IO ImportUrl                  | The URL to the IOImport, Example: "https://<.ioservice.net/importws/import"  | Yes       |
| IO ImportUsername             | The UserName to connect to the IO Import                                     | Yes       |
| IO ImportPassword             | The Password to connect to the IO Import                                     | Yes       |
| CSV Mapping Deskundigheid (Education)        | The Path to the mapping file for Deskundigheid (education)                                       | Yes       |
| CSV Separation Character      | Mapping File CSV Separation Character                                        | Yes       |
| Encoding for the csv imports. | Default value is 'Default'                                                   | Yes       |

### Correlation configuration

The correlation configuration is used to specify which properties will be used to match an existing account within _NedapOns-Employee_ to a person in _HelloID_.   

This Correlation setting is not used by this connector, so disable this.

| Setting                   | Value  |
| ------------------------- | ------ |
| Enable correlation        | `false` |
| Person correlation field  | `-`    |
| Account correlation field | `-`    |

> [!TIP]
> _For more information on correlation, please refer to our correlation [documentation](https://docs.helloid.com/en/provisioning/target-systems/powershell-v2-target-systems/correlation.html) pages_.

### Available lifecycle actions

The following lifecycle actions are available:

| Action             | Description                                                                                                                                                                                                                                                                                           |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| create.ps1         | Updates the associated Nedap accounts based on the contracts in condition from the Business Rules.                                                                                                                                                       |
| configuration.json | Contains the connection settings and general configuration for the connector.                                                                                                                                                                                                                         |
| fieldMapping.json  | Defines mappings between Helloid (person) fields and target system account fields.                                                                                                                                                                                                                        |

### Field mapping

The field mapping can be imported using the `_fieldMapping.json` file.

#### The Primary Contract Calculation foreach employment
The primary contract calculation is the same as the HelloID primary contract calculation. Note that the order of properties can be adjusted to meet the customer's requirements.

```Powershell
## Configuration
# Primary Contract Calculation foreach employment
$firstProperty =  @{ Expression = { $_.Details.Fte };             Descending = $true  }
$secondProperty = @{ Expression = { $_.Details.HoursPerWeek };    Descending = $true  }
$thirdProperty =  @{ Expression = { $_.Details.Sequence };        Descending = $true  }
$fourthProperty = @{ Expression = { $_.EndDate };                 Descending = $true  }
$fifthProperty =  @{ Expression = { $_.StartDate };               Descending = $false }
$sixthProperty =  @{ Expression = { $_.ExternalId };              Descending = $false }

# Priority Calculation Order (High priority -> Low priority)
$splatSortObject = @{
    Property = @(
        $firstProperty,
        $secondProperty,
        $thirdProperty,
        $fourthProperty,
        $fifthProperty,
        $sixthProperty)
}
```

#### Script Mapping lookup values
```Powershell
# Lookup value which is used in the mapping to determine the education (Deskundigheid) 
$educationPrimaryLookupKey   = { $_.Title.ExternalId }         # Mandatory
$educationSecondaryLookupKey = { $_.Department.ExternalId } # Not Mandatory
```

## Remarks

- This connector only correlates a HelloID person with one or more employee accounts in NedapONS and updates the education (deskundigheid). It updates the education of all the accounts from the contracts in the conditions.
- The script used for updating is identical to the script used for creating.

### Correlation

Because Nedap contains multiple accounts per HelloID person (contract), accounts are correlated based on a custom contract property called `custom.NedapOnsIdentificationNo`. An example of this can be found in the [Prerequisites](#prerequisites) section. This approach is necessary because HelloID does not support correlation based on contract properties.

### Primary Contract

Since the connector supports multiple accounts for a single HelloID person (per contract), the default primary contract calculation may not always be applicable. The connector includes an example of a primary contract calculation that determines the primary contract for each employment in Nedap.

### IO Import validation

When updating an employee account, it's not possible to directly verify whether the _Deskundigheid_ has been successfully updated in Nedap. To confirm this, you need to check the_ Import Report_ in the Nedap UI by going to: _Beheer > Import > Importrapportage inzien._

### OutputInfo

In the field mapping, some properties have a `_outputInfo` prefix. These properties are **only** used to return data from the connector, and the information is visible in the account data or in the notification. Since there can be multiple accounts, the connector returns the values as comma-separated strings. Note that there is **no link** between the individual values.

## Fact Sheet

The following table displays an overview of the functionality for the Nedap Ons connector for HelloID Provisioning and Service Automation.

| Type of action                      | Nedap | HelloID provisioning                                            | HelloID Service Automation |
| ----------------------------------- | ----- | --------------------------------------------------------------- | -------------------------- |
| Create Employees                    | Yes   | No, *Only the EducationId (Deskundigheid) will be set.*  no scheduling* | No                         |
| Update Employees                    | Yes   | No, *Only the EducationId (Deskundigheid) will be set.*  no scheduling* | No                         |


## Development resources

### API endpoints

The following endpoints are used by the connector

| Endpoint                               | Description                   | Type     |
| -------------------------------------- | ----------------------------- | -------- |
| /t/employees/by_identification_no/<id> | Retrieve Employee Information | Rest     |
| /importws/import                       | CRUD Employee Information     | IOImport |

### API documentation

* Nedap API documentation → [klik](https://www.ons-api.nl/english/technical/APIS.html)
* Nedap XML IOImport Manual → [klik](https://support.nedap-ons.nl/support/solutions/articles/103000265750-gegevens-importeren-via-xml-bestanden)
* Nedap XML IOImport Documentation → [klik](https://hc-freshdesk-assets.s3.eu-central-1.amazonaws.com/Supportportaal%20Ons%20Suite/Technische%20documenten/XML%20Interface/io_server_xml_interface.pdf)
* Nedap ONS authorization Manual → [klik](https://www.ons-api.nl/english/authorization/AuthorizationInOns.html)

## Getting help

> [!TIP]
> _For more information on how to configure a HelloID PowerShell connector, please refer to our [documentation](https://docs.helloid.com/en/provisioning/target-systems/powershell-v2-target-systems.html) pages_.

> [!TIP]
>  _If you need help, feel free to ask questions on our [forum](https://forum.helloid.com/forum/helloid-connectors/provisioning/1297-helloid-prov-target-nedap-ons-deskundigheid)_.

## HelloID docs

The official HelloID documentation can be found at: https://docs.helloid.com/