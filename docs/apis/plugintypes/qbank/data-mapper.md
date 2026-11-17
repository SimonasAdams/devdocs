---
title: Data mapper API
tags:
  - Question bank
  - Plugins
  - Import
  - Export
---

<Since version="6.0" issueNumber="MDL-78520" />

Question bank plugins (`qbank_*`) can store extra data against a question, such as custom fields, review
states or links to external resources. The data mapper API lets a plugin include that data when
questions are exported, and restore it when they are imported, so it is not lost when questions move
between courses or sites.

At the moment the Moodle XML format (`qformat_xml`) is the only format that reads and writes plugin
data. Other formats ignore it.

## How it works

When questions are **exported**:

1. `qformat_default::exportprocess()` asks every enabled qbank plugin for its data, with one call per
   plugin for all of the questions being exported.
2. Each plugin returns its data as a list of named *nodes*.
3. `qformat_xml` writes each plugin's nodes in a `<qbank>` element inside the question.

When questions are **imported**:

1. `qformat_xml` reads the `<qbank>` element back into nodes.
2. After a question, its options and its tags have been saved, `qformat_default::importprocess()`
   passes each enabled plugin its own nodes for that question.
3. If a plugin throws a `moodle_exception`, the import of that question is rolled back and the
   exception message is shown to the user.

Data is only exported and imported for **enabled** plugins. If a plugin is disabled or not installed
on the importing site, its data in the file is ignored.

## Implementing a data mapper

### 1. Return a data mapper from your plugin feature

Override `get_data_mapper()` in your plugin's `plugin_feature` class. The default implementation
returns a `data_mapper_base`, which exports and imports nothing.

```php title="question/bank/example/classes/plugin_feature.php"
namespace qbank_example;

use core_question\local\bank\data_mapper_base;
use core_question\local\bank\plugin_features_base;

class plugin_feature extends plugin_features_base {

    #[\Override]
    public function get_data_mapper(): data_mapper_base {
        return new data_mapper();
    }
}
```

### 2. Extend `data_mapper_base`

Override the two methods of `\core_question\local\bank\data_mapper_base`:

| Method | Called | Returns |
|---|---|---|
| `get_question_data(array $questionids): array` | On export, once for all exported questions | `[$questionid => array $nodes]`, with a key for **every** id passed in, even if there is no data for it |
| `save_question_data(int $questionid, array $data): void` | On import, once per imported question that has data for your plugin | Nothing. Throw a `moodle_exception` if the data can't be saved |

Start from `parent::get_question_data($questionids)`, which returns an empty list for each question,
and add your nodes to it.

## The data format

A plugin's data for a question is a **list of nodes**. Each node is an array with a `name`, and is
either a leaf or a branch.

### Leaf nodes

A leaf holds a single value, as formatted text:

| Key | Type | Notes |
|---|---|---|
| `name` | `string` | Required. |
| `text` | `string` | Required. The value. |
| `format` | `int` | Optional `FORMAT_*` constant. Defaults to `FORMAT_PLAIN`. |
| `files` | `\stored_file[]` | **Export only**, optional. Files embedded in the text. |
| `itemid` | `int` | **Import only**. A draft area holding the embedded files. Only set if the file contained some. |

### Branch nodes

A branch groups other nodes:

| Key | Type | Notes |
|---|---|---|
| `name` | `string` | Required. |
| `children` | `array` | A non-empty list of nodes. |

### Lists and records

Names don't need to be unique, so you can represent a list by repeating a name, and a record by using
a branch. For example, a plugin storing a review status and a list of sources could return:

```php
[
    ['name' => 'status', 'text' => 'approved'],
    ['name' => 'source', 'children' => [
        ['name' => 'title', 'text' => 'Physics textbook, chapter 3'],
        ['name' => 'notes', 'text' => '<p>Adapted from exercise 3.2.</p>', 'format' => FORMAT_HTML],
    ]],
    ['name' => 'source', 'children' => [
        ['name' => 'title', 'text' => 'Past exam paper, 2024'],
    ]],
]
```

>***Exported files can be imported on a different site. Export values that mean the same thing everywhere,
>such as usernames, idnumbers or labels, rather than database ids or other site-specific values. On
>import, look the value up again on the current site.***


## Example

```php title="question/bank/example/classes/data_mapper.php"
namespace qbank_example;

class data_mapper extends \core_question\local\bank\data_mapper_base {

    #[\Override]
    public function get_question_data(array $questionids): array {
        global $DB;
        $questiondata = parent::get_question_data($questionids);
        [$insql, $params] = $DB->get_in_or_equal($questionids);
        foreach ($DB->get_records_select('qbank_example', "questionid {$insql}", $params) as $record) {
            $questiondata[$record->questionid][] = ['name' => 'status', 'text' => $record->status];
            $questiondata[$record->questionid][] = [
                'name' => 'notes',
                'text' => $record->notes,
                'format' => (int) $record->notesformat,
            ];
        }
        return $questiondata;
    }

    #[\Override]
    public function save_question_data(int $questionid, array $data): void {
        global $DB;
        $record = (object) ['questionid' => $questionid];
        foreach ($data as $node) {
            if (isset($node['children'])) {
                continue;
            }
            if ($node['name'] === 'status') {
                $record->status = $node['text'];
            } else if ($node['name'] === 'notes') {
                $record->notes = $node['text'];
                $record->notesformat = $node['format'] ?? FORMAT_PLAIN;
            }
        }
        if (!in_array($record->status ?? '', ['draft', 'approved'])) {
            throw new \moodle_exception('invalidstatus', 'qbank_example');
        }
        $DB->insert_record('qbank_example', $record);
    }
}
```

## Embedded files

Rich text often contains embedded files, using `@@PLUGINFILE@@` placeholders.

- **Export:** put the raw stored text, with its placeholders, in `text`, and the files in `files`.
  The files are written into the XML as base64, inside the node, in the same way as question text
  images. The export file is self-contained, so it can be imported on any site.
- **Import:** the files are decoded into a draft area owned by the importing user, and its id is passed
  in `itemid`. Move them into your own file area, for example with `file_save_draft_area_files()` or
  `file_postupdate_standard_editor()`. The `@@PLUGINFILE@@` placeholders in `text` are kept.

```php
// Export.
$fs = get_file_storage();
$node = [
    'name' => 'notes',
    'text' => $record->notes,
    'format' => (int) $record->notesformat,
    'files' => $fs->get_area_files($contextid, 'qbank_example', 'notes', $record->id, 'filepath, filename', false),
];

// Import.
if (!empty($node['itemid'])) {
    $notes = file_save_draft_area_files($node['itemid'], $contextid, 'qbank_example', 'notes', $recordid,
        null, $node['text']);
}
```

## Errors and warnings

- **Fatal for the question:** throw a `moodle_exception` (or a subclass, such as a `dml_exception`)
  from `save_question_data()`. The import of that question is rolled back, the message is shown and the
  import stops. Other PHP errors are not caught.
- **Not fatal:** if some of the data can't be used but the question should still be imported, skip it
  and tell the user with `\core\notification::warning()`. Use a language string, and escape any values
  taken from the file with `s()`.

## In Moodle XML

Each plugin's data is written in a `<qbank>` element, inside an element named after the plugin
(without the `qbank_` prefix). Every node is an `<item>` with a `name` attribute. A leaf has a `format`
attribute, a `<text>` element and any `<file>` elements, like question text. A branch contains further
`<item>` elements.

```xml
<question type="truefalse">
  ...
  <qbank>
    <example>
      <item name="status" format="plain_text">
        <text>approved</text>
      </item>
      <item name="notes" format="html">
        <text><![CDATA[<p><img src="@@PLUGINFILE@@/a.png"></p>]]></text>
<file name="a.png" path="/" encoding="base64">iVBORw0KGgo...</file>
      </item>
      <item name="source">
        <item name="title" format="plain_text">
          <text>Physics textbook, chapter 3</text>
        </item>
      </item>
    </example>
  </qbank>
</question>
```

## Reference implementation: question custom fields

`qbank_customfields` uses this API (`\qbank_customfields\data_mapper`):

- Each custom field is exported as a leaf named after the field's shortname. Its value comes from
  `\core_customfield\data_controller::prepare_export_value()`, so a select field is exported as the
  option label rather than its position. Embedded textarea files come from `get_files()`.
- On import, values are matched to the site's custom fields by shortname, and saved through
  `prepare_import_value()` and the handler's `instance_form_save()`. A value is skipped, with a warning,
  if the field doesn't exist, the user can't edit it, or the value isn't valid for it (for example, an
  unknown select option).

Custom field types can support this by overriding `prepare_export_value()`, `get_files()` and
`prepare_import_value()` on their `data_controller`. `customfield_select` and `customfield_textarea` do
this.
