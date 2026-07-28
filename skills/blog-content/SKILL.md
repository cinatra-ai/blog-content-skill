---
name: blog-content
description: Use when the user asks about the blog dashboard, blog projects/ideas/posts, drafting/editing blog content, or running the blog-content publish workflow. Names the new operator surface (project→idea→post selection chain) and the publish-launcher portlet.
metadata:
  # cinatra-watches: the workflow-instantiate primitive + the blog-content publish
  # packages this skill's publish-workflow instructions depend on (cinatra#188).
  cinatra-watches:
    primitives:
      - workflow_template_instantiate
    packages:
      - "@cinatra-ai/blog-content-workflow"
      - "@cinatra-ai/blog-pipeline-agent"
      - "@cinatra-ai/blog-wordpress-publish-agent"
---

You are guiding the operator through the **Blog Content** surface that ships with the
`@cinatra-ai/blog-content-workflow` extension. The dashboard is materialized per Cinatra project
(`/dashboards/[id]`) and composes nine portlets in a strict selection chain:

```
projects-pane → project-detail
              → ideas-pane → posts-pane → draft-editor
                                        → hero-image
                                        → version-history
                                        → publish-launcher → publish-status
```

## Selection chain (left-to-right)

- **projects-pane** — lists `@cinatra-ai/assets:blog-project` objects. Operator picks a project; its `selectedId` becomes the parent for **ideas-pane**.
- **ideas-pane** — lists `@cinatra-ai/assets:blog-idea` objects under the selected project. Its `selectedId` becomes the parent for **posts-pane**.
- **posts-pane** — lists `@cinatra-ai/assets:blog-post` objects under the selected idea. Its `selectedId` flows downstream to:
  - **draft-editor** — inline markdown editor over `postArtifactId` (ref-swap via `blog_post_update`).
  - **hero-image** — current image artifact preview (`imageArtifactId`); regen lands in a future phase.
  - **version-history** — the `postArtifactId` ref-swap timeline (only events that changed the field).

## Publish workflow

- **publish-launcher** wraps `workflow_template_instantiate` for the `blog-content-workflow` BPMN. It renders typed pickers from the template's placeholder hints:
  - `projectId` (hint `blog-project`) — object-list picker over blog projects.
  - `postId` (hint `blog-post`) — object-list picker over blog posts.
  - `wordpressInstanceId` (hint `wordpress-instance`) — picker backed by `wordpress_instances_list` (credentials stripped server-side).
- The workflow steps are: `review_publish_bundle` (approval) → `create_wordpress_draft` (agent: `@cinatra-ai/blog-wordpress-publish-agent`) → `publish_in_wordpress_admin` (manual) → `notify_publish_checkpoint_complete`.
- **publish-status** shows the project's workflows + their statuses (read-only summary).

## Idempotency note

`blog_post_publish_wordpress_start` short-circuits with an `idempotentNoop` envelope when a non-deleted WordPress draft already exists for `{projectId, postId, wordpressInstanceId}`; the existing draft refs (`wordpressDraftId`, `wordpressPostId`) are returned and no new job is enqueued.

## What this skill is NOT

- It is NOT the blog AGENT (`@cinatra-ai/blog-pipeline-agent`); it is the operator workspace.
- It does NOT start/approve workflows on behalf of the user — `chat-workflow-authoring` owns proposal authoring, the dashboard owns execution.
