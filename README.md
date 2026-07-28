# Cinatra Blog Content Skill

The operator's guide to Cinatra's Blog Content dashboard: it walks an assistant through the project→idea→post selection chain, the draft and hero-image panes, and how a post reaches WordPress through the publish launcher. Packaged as its own skill so every assistant that lists it in its skill bundle resolves it from the skills catalog instead of an embedded copy.

**Install:** Install `@cinatra-ai/blog-content-skill` in your Cinatra instance. Assistants that list the `blog-content` slug in their skill bundle pick it up from the catalog.

**Usage:** The skill is mounted into assistant conversations on demand — you do not invoke it directly. It answers questions about the blog dashboard and names the exact portlets and workflow steps involved in publishing.

**Configuration:** None. The skill carries no credentials and reads no settings; the dashboard extension supplies the surfaces it describes.

**Development:** Clone the repository and run `node extension-kind-gate.mjs --package-root .` to validate the manifest. The bundle lives in `skills/blog-content/`.

**Troubleshooting:** If the assistant cannot answer blog-dashboard questions, the bundle is not mounted — check that the assistant's skill bundle lists `blog-content`. If publish guidance drifts from the real steps, compare against the current `@cinatra-ai/blog-content-workflow` template.

## Works with

- Cinatra chat, WordPress authoring, and Drupal authoring assistant skill bundles
- The Cinatra blog-content workflow extension, whose dashboard and publish flow this skill describes

## Capabilities

- Guide the operator through the project→idea→post selection chain and its nine portlets
- Explain the draft-editor, hero-image, and version-history panes over the selected post
- Walk through launching the publish workflow with its typed project, post, and instance pickers
- Describe the review→draft→manual-publish→notify step sequence and the read-only status pane
- State the idempotent short-circuit when a WordPress draft already exists for the same triple
