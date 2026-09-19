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
  - `wordpressInstanceId` (hint `wordpress-instance`) — picker over the operator's connected WordPress instances (credentials stripped server-side); the per-operation connector tool that used to back it was removed in the connector's open-catalog cutover (wordpress-mcp-connector#101) — content operations against the selected instance now go through the generic `wordpress_site_tools_list` / `wordpress_site_tool_call` catalog primitives instead.
- The workflow steps are: `review_publish_bundle` (approval) → `create_wordpress_draft` (agent: `@cinatra-ai/blog-wordpress-publish-agent`) → `publish_in_wordpress_admin` (manual) → `notify_publish_checkpoint_complete`. Those four step ids are the template's own and are unchanged; what the second step actually does depends on the installed version of the publish agent.
  - `@cinatra-ai/blog-wordpress-publish-agent` 0.4.x — the agent takes an artifact reference, never raw text and never a blog record: the post artifact and the exact representation revision the operator continued with. It pauses at its own `draft-confirm` HITL gate so the operator confirms BEFORE the post goes to the site, then creates the page PUBLICLY through the site's own ability catalog (`wordpress_site_tools_list` to find the ability, `wordpress_site_tool_call` to invoke it — an ability whose only outcome is a draft does not satisfy the step, because a draft has no public address), and hands back the page's address, which a deterministic write-back step merges onto the post artifact's own data as `wordpressPublishedUrl`, `wordpressPublishedExternalId` and `wordpressPublishedRevisionId`. It creates no WordPress draft and reaches no blog record, so `publish_in_wordpress_admin` has nothing left to publish by hand. One mismatch to raise with the operator: the template hands this step `projectId`, `postId` and `wordpressInstanceId`, which are not the three inputs 0.4.x requires (`postArtifactId`, `postRepresentationRevisionId`, `wordpressInstanceId`) — tell the operator to reconcile the template with the installed agent version rather than promising this sequence reaches a published page.
  - `@cinatra-ai/blog-wordpress-publish-agent` 0.3.x — the step did create a WordPress draft from `projectId`, `postId` and `wordpressInstanceId`, and `publish_in_wordpress_admin` was where a person published that draft by hand in the WordPress admin.
- **publish-status** shows the project's workflows + their statuses (read-only summary).

## Re-running a publish

- `@cinatra-ai/blog-wordpress-publish-agent` 0.4.x — there is no draft-level short-circuit, because the publish step reaches no blog record and calls no host publish primitive at all: it does not use `blog_post_publish_wordpress_start`, so that primitive's `idempotentNoop` guard does not cover this path. The only thing between a re-run and a second public page is the confirmation the agent asks before the site write — a decision the operator makes, not an automatic check — so read the address written back onto the post artifact (`wordpressPublishedUrl`, `wordpressPublishedExternalId`, `wordpressPublishedRevisionId`) before confirming: that address is what tells an operator the post is already out.
- `@cinatra-ai/blog-wordpress-publish-agent` 0.3.x — `blog_post_publish_wordpress_start` short-circuits with an `idempotentNoop` envelope when a non-deleted WordPress draft already exists for `{projectId, postId, wordpressInstanceId}`; the existing draft refs (`wordpressDraftId`, `wordpressPostId`) are returned and no new job is enqueued. That primitive is still part of the host — the 0.4.x publish step simply does not use it.

## What this skill is NOT

- It is NOT the blog AGENT (`@cinatra-ai/blog-pipeline-agent`); it is the operator workspace.
- It does NOT start/approve workflows on behalf of the user — `chat-workflow-authoring` owns proposal authoring, the dashboard owns execution.
