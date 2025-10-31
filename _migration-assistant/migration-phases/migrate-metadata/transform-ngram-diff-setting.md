---
layout: default
title: Transform ngram diff setting
nav_order: 5
parent: Migrate metadata
grand_parent: Migration phases
permalink: /migration-assistant/migration-phases/migrate-metadata/transform-ngram-diff-setting/
---

# Transform ngram diff setting

{: .note }
This transformation may not apply to your use case, but the framework for creating a transformation is designed to handle mutations, data enrichments, and other modifications when modifying workloads or moving them to a new target.

This guide explains how to configure Migration Assistant to add the `max_ngram_diff` setting to indexes and templates that use ngram analyzers during migration to newer OpenSearch versions.

## Overview

The `max_ngram_diff` setting was introduced in later versions of OpenSearch to limit the difference between `min_gram` and `max_gram` values in ngram tokenizers and filters. This setting prevents potential performance issues and memory consumption problems that can occur with large ngram ranges.

When migrating from earlier Elasticsearch versions to OpenSearch, you can configure Migration Assistant to detect indexes and templates that use ngram analyzers and add the `max_ngram_diff` setting with a value of 30 to ensure compatibility.

## Configure ngram diff transformation

You can customize how ngram analyzers are handled during metadata migrations by supplying a transformation configuration file using the following steps:

1. Open the Migration Assistant console.
2. Create a JavaScript file to define your transformation logic using the following command:

   ```bash
   vim /shared-logs-output/ngram-diff-setting.js
   ```
   {% include copy.html %}

3. Add the ngram transformation logic to handle ngram analyzers. For an example implementation, see the [example `ngram-diff-setting.js` implementation](#example-ngram-diff-settingjs-implementation).
4. Create a transformation descriptor file using the following command:

   ```bash
   vim /shared-logs-output/transformation.json
   ```
   {% include copy.html %}

5. Add a reference to your JavaScript file in `transformation.json`.
6. Run the metadata migration and supply the transformation configuration using a command similar to the following:

   ```bash
   console metadata migrate \
     --transformer-config-file /shared-logs-output/transformation.json
   ```
   {% include copy.html %}

### Example `ngram-diff-setting.js` implementation

The following script demonstrates how to detect ngram analyzers and add the `max_ngram_diff` setting:

```javascript
function main(context) {
  const NGRAM_DIFF_ALLOWED = 30;

  const containsNGram = (analysis) => {
    if (!analysis || typeof analysis !== "object") return false;
    
    for (const sectionKey of ["tokenizer", "filter"]) {
      const section = analysis[sectionKey];
      if (!section || typeof section !== "object") continue;
      
      for (const [, cfg] of Object.entries(section)) {
        const type = cfg && typeof cfg === "object" ? cfg.type : undefined;
        if (typeof type === "string" && type.toLowerCase().includes("ngram")) {
          return true;
        }
      }
    }
    return false;
  };

  const processDocument = (doc) => {
    if (!doc || !doc.type || !doc.body) return doc;
    
    const isTemplate = ["template", "index_template", "component_template"].includes(doc.type);
    const isIndex = doc.type === "index";
    
    if (!isTemplate && !isIndex) return doc;
    
    const settings = doc.body.settings;
    if (!settings || typeof settings !== "object") return doc;
    
    const analysis = settings.analysis || (settings.index && settings.index.analysis);
    if (containsNGram(analysis)) {
      if (!settings.index) settings.index = {};
      settings.index.max_ngram_diff = NGRAM_DIFF_ALLOWED;
    }
    
    return doc;
  };

  return (document) => {
    if (Array.isArray(document)) {
      return document.map(processDocument);
    }
    return processDocument(document);
  };
}
(() => main)();
```
{% include copy.html %}

### Example `transformation.json`

The following JSON file references your transformation script:

```json
[
  {
    "JsonJSTransformerProvider": {
      "initializationScriptFile": "/shared-logs-output/ngram-diff-setting.js",
      "bindingsObject": "{}"
    }
  }
]
```
{% include copy.html %}

## Transformation behavior

<table style="border-collapse: collapse; border: 1px solid #ddd;">
  <thead>
    <tr>
      <th style="border: 1px solid #ddd; padding: 8px;">Source configuration</th>
      <th style="border: 1px solid #ddd; padding: 8px;">Target configuration</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="border: 1px solid #ddd; padding: 8px;">
        <pre><code>{
  "settings": {
    "analysis": {
      "filter": {
        "ngram_filter": {
          "type": "ngram",
          "min_gram": 2,
          "max_gram": 20
        }
      }
    }
  }
}</code></pre>
      </td>
      <td style="border: 1px solid #ddd; padding: 8px;">
        <pre><code>{
  "settings": {
    "index": {
      "max_ngram_diff": "30"
    },
    "analysis": {
      "filter": {
        "ngram_filter": {
          "type": "ngram",
          "min_gram": 2,
          "max_gram": 20
        }
      }
    }
  }
}</code></pre>
      </td>
    </tr>
    <tr>
      <td style="border: 1px solid #ddd; padding: 8px;">
        <pre><code>{
  "settings": {
    "index": {
      "number_of_shards": 5
    },
    "analysis": {
      "tokenizer": {
        "my_tokenizer": {
          "type": "edge_ngram",
          "min_gram": 1,
          "max_gram": 15
        }
      }
    }
  }
}</code></pre>
      </td>
      <td style="border: 1px solid #ddd; padding: 8px;">
        <pre><code>{
  "settings": {
    "index": {
      "number_of_shards": 5,
      "max_ngram_diff": "30"
    },
    "analysis": {
      "tokenizer": {
        "my_tokenizer": {
          "type": "edge_ngram",
          "min_gram": 1,
          "max_gram": 15
        }
      }
    }
  }
}</code></pre>
      </td>
    </tr>
  </tbody>
</table>

## Detection logic

The transformation script scans for ngram usage in the following locations:
- **Tokenizers**: `ngram`, `edge_ngram`, or any tokenizer type containing "ngram"
- **Filters**: `ngram`, `edge_ngram`, or any filter type containing "ngram"
- **Analysis settings**: Both at `settings.analysis` and `settings.index.analysis` levels

## Affected components

This transformation applies to:
- **Index templates**: Legacy templates with ngram analyzers
- **Index creation requests**: Indexes using ngram tokenizers or filters
- **Component templates**: Templates containing ngram analysis configurations

## Performance considerations

The `max_ngram_diff` setting helps prevent:
- Excessive memory usage from large ngram ranges
- Performance degradation during indexing and search operations
- Potential cluster instability from resource exhaustion

The transformation sets `max_ngram_diff` to 30, which allows for ngram ranges up to 30 characters difference between `min_gram` and `max_gram` values.

## Troubleshooting

If you encounter issues with ngram diff setting transformation:

1. **Verify ngram usage** -- Check if your indexes or templates actually use ngram analyzers:
   ```bash
   GET /your-index/_settings
   GET /_template/your-template
   ```

2. **Check migration logs** -- Review the detailed migration logs for any warnings or errors:
   ```bash
   tail /shared-logs-output/migration-console-default/*/metadata/*.log
   ```

3. **Validate setting application** -- After migration, verify that the `max_ngram_diff` setting has been correctly added:
   ```bash
   GET /your-index/_settings
   ```

4. **Review ngram ranges** -- Ensure that your ngram configurations don't exceed the 30-character difference limit. If they do, you may need to adjust your analyzer configurations.

## Summary

By using a transformation configuration, you can automatically add the `max_ngram_diff` setting to indexes and templates that use ngram analyzers during metadata migration. This ensures that your target OpenSearch cluster can handle ngram configurations without performance issues.

## Related documentation

- [Transform field types documentation]({{site.url}}{{site.baseurl}}/migration-assistant/migration-phases/migrate-metadata/handling-field-type-breaking-changes/) -- Configure custom field type transformations.
- [Analysis documentation]({{site.url}}{{site.baseurl}}/analyzers/) -- Learn about text analysis in OpenSearch.
- [Ngram tokenizer documentation]({{site.url}}{{site.baseurl}}/analyzers/tokenizers/ngram/) -- Learn about ngram tokenizers.