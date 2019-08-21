# Import data from agency annual reports in NIEM-XML format

This module has the following components:

- `config/install/*.yml`: configuration files defining custom migrations
- `src/Form/AgencyXmlUploadForm.php`: define a custom form for uploading the XML
  report
- `Plugin/migrate/process/*.php`: custom process plugins used by the migrations

After making changes in the `config/install/` directory, the usual workflow is

```
drush cr
drush cim --partial --source=modules/custom/foia_upload_xml/config/install
drush cex
```

When testing, you can repeat the first two steps as needed and only export
configuration when you are ready to commit.

This workflow creates two nearly identical copies of the configuration files:
on in the module's `config/install/` directory and one in the site's
configuration directory. There are at least two advantages to this redundancy:

1. We can add comments to the files in the module. Comments are stripped from
   the files generated by `drush cex`.
2. It provides a safeguard in case someone else deletes files in the site
   configuration directory.

## Running the migrations

One option is to run the migrations through the admin UI. The upload form at
`/report/upload` has a link to the relevant page, and it also redirects there
after uploading the file.

The other option is to use the command line:

```
drush mim --group=foia_xml --update
```

Use `drush ms --group=foia_xml` to check the status of the migrations. To see
available options, use `drush help mim`.

## Custom migrations

There are a lot of files in the `/config/install/` directory, but it is not
too complicated once you understand how they are organized.

### Beginning and end

The file `migrate_plus.migration_group.foia_xml.yml` defines the `foia_xml`
migration group. All the custom migrations belong to this group.

The file `migrate_plus.migration.foia_agency_report.yml` creates a single
`annual_foia_report_data` node from the XML report. It depends on all the
other migrations, so it will only run when they are complete.

The XML file has a section that associates internal identifiers to each of the
component abbreviations. It looks something like this:

```
  <nc:Organization s:id="ORG0">
    <nc:OrganizationAbbreviationText>USDA</nc:OrganizationAbbreviationText>
    <nc:OrganizationName>United States Department of Agriculture</nc:OrganizationName>
    <nc:OrganizationSubUnit s:id="ORG1">
      <nc:OrganizationAbbreviationText>AMS</nc:OrganizationAbbreviationText>
      <nc:OrganizationName>Agricultural Marketing Service</nc:OrganizationName>
    </nc:OrganizationSubUnit>
    ...
  </nc:Organization>
```

In this example, the internal identifiers "ORG0" and "ORG1" are associated to
the agency (USDA) and one of its components (AMS), respectively.

The file `migrate_plus.migration.component.yml` defines a migration that does
not create any entities. It just creates a map from the internal identifier to
the corresponding `agency_component` node. Other migrations use this map via
the `migration_lookup` process plugin, so all the other migrations depend on
this one.

More precisely, the `component` migration creates a map from the three keys
report year, agency abbreviation, internal identifier to the node ID of the
corresponding `agency_component` node. The first two keys are used to identify
the current XML report, or the corresponding `annual_foia_report_data` node.

### The middle

For each paragraph field (and associated "overall" fields) we add two
migrations.

The first is similar to the `component` migration, and depends directly on it.
For example, Section V.A of the report contains a section like this:

```
  <foia:ProcessedRequestSection>
    ...
    <foia:ProcessingStatisticsOrganizationAssociation>
      <foia:ComponentDataReference s:ref="PS19" />
      <nc:OrganizationReference s:ref="ORG19" />
    </foia:ProcessingStatisticsOrganizationAssociation>
    <foia:ProcessingStatisticsOrganizationAssociation>
      <foia:ComponentDataReference s:ref="PS0" />
      <nc:OrganizationReference s:ref="ORG0" />
    </foia:ProcessingStatisticsOrganizationAssociation>
    ...
  </foia:ProcessedRequestSection>
```

Withing this section, "PS0" and "PS19" are associated to the internal
identifiers "ORG0" and "ORG19", respectively: these are the identifiers
handled by the `component` migration. The file
`migrate_plus.migration.component_requests_va.yml` is used to map these
section-specific identifiers "PS0" and "PS19" and so on to the corresponding
`agency_component` nodes.

The second migration for this section is defined by the file
`migrate_plus.migration.foia_requests_va.yml`. This migration creates
Paragraphs of type `foia_req_va`. These Paragraphs are then referenced in
`field_foia_requests_va` in the `foia_agency_report` migration, using the
`migration_lookup` process plugin.

## Handling another section

Each section of the annual report has a corresponding section of the XML file,
and we should be able to handle them all as described in the previous section.
There are some nested Paragraphs, and handling those will be a little
different.

In addition to adding two migrations, you will have to update the
`foia_agency_report` migration:

1. Add fields in the `source/fields` section.
2. Map those fields in the `process` section. Remember that most destination
   fields have the prefix `field_`.
3. Use the `migration_lookup` process plugin, referencing your Paragraph
   migration, to populate the relevant field.
4. Add your Paragraph migration to the list of dependencies at the end of the
   file.

Step 1 is a little tricky, since you have to find the correct XPath
expressions to extract the data from the XML. The "overall" fields are closely
related to the corresponding per-component fields. For example,
`migrate_plus.migration.foia_requests_va.yml` includes the following:

```
source:
  item_selector: '/iepd:FoiaAnnualReport/foia:ProcessedRequestSection/foia:ProcessingStatistics'
  fields:
    -
      name: field_req_pend_start_yr
      label: 'Requests pending at the start of the year'
      selector: 'foia:ProcessingStatisticsPendingAtStartQuantity'
```

Combining the `item_selector` and the selector for this field gives (adding
line breaks for readability)

```
  /iepd:FoiaAnnualReport
  /foia:ProcessedRequestSection
  /foia:ProcessingStatistics
  /foia:ProcessingStatisticsPendingAtStartQuantity
```

Compare this with the corresponding lines in
`migrate_plus.migration.foia_agency_report.yml`:

```
source:
  item_selector: '/iepd:FoiaAnnualReport'
  fields:
    -
      name: overall_req_pend_start_yr
      label: 'Overall requests pending at the start of the year'
      selector: 'foia:ProcessedRequestSection/foia:ProcessingStatistics[@s:id="PS0"]/foia:ProcessingStatisticsPendingAtStartQuantity'
```

Combining these two selectors gives

```
  /iepd:FoiaAnnualReport
  /foia:ProcessedRequestSection
  /foia:ProcessingStatistics[@s:id="PS0"]
  /foia:ProcessingStatisticsPendingAtStartQuantity
```

The only difference is the additional selector `[@s:id="PS0"]`.

The third part (migration lookup) is the most complicated, so let's break it
down. The process pipeline starts with

```
  field_foia_requests_va:
    -
      plugin: foia_array_pad
      source: component_va
      prefix:
        - report_year
        - agency
```

The source field `component_va` is an array of strings, like

```
["PS1", "PS2", ... ]
```

The `foia_array_pad` plugin is custom: it adds the source fields listed under
`prefix`, producing an array of arrays:

```
[[2018, "USDA", "PS1"], [2018, "USDA", "PS2"], ... ]
```

The next step in the process pipeline is to apply `migration_lookup` to each
of those triples:

```
    -
      plugin: sub_process
      process:
        combined:
          plugin: migration_lookup
          source:
            - '0'
            - '1'
            - '2'
          migration:
            - foia_requests_va
          no_stub: true
```

This results in an array of arrays. The inner arrays have a single key,
`combined`, and the corresponding value is the result of applying
`migration_lookup` for the `foia_requests_va` migration to the triple from the
preceding step. Since that is a Paragraph migration, this value is an array of
two numbers (entity ID and revision ID).

Still within the`sub_process` plugin, we have

```
        target_id:
          plugin: extract
          source: '@combined'
          index:
            - '0'
        target_revision_id:
          plugin: extract
          source: '@combined'
          index:
            - '1'
```

At this point, we have an array of arrays. One of the inner arrays looks
something like this:

```
  [
    'combined' => [123, 456],
    'target_id' => 123,
    'target_revision_id' => 456,
  ]
```

This field expects a `target_id` and a `target_revision_id`, and the
`combined` value is ignored.