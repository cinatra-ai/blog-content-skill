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
  - **hero-image** — draws no image and offers no regeneration: the host has no card for its kind (`artifact-edit-binary-prompt`), so it draws only its header (instanceId and kind@version) and the line "Portlet kind artifact-edit-binary-prompt is not yet available." The post's featured image is the plain image artifact (`@cinatra-ai/image-artifact:image`), drawn by the image display on its artifact page, the review step and the review card; never promise an image preview or a remake of the image on this dashboard.
  - **version-history** — the `postArtifactId` ref-swap timeline (only events that changed the field).

## Publish workflow

- **publish-launcher** wraps `workflow_template_instantiate` for the `blog-content-workflow` BPMN. It renders typed pickers from the template's placeholder hints:
  - `projectId` (hint `blog-project`) — object-list picker over blog projects.
  - `postId` (hint `blog-post`) — object-list picker over blog posts.
  - `wordpressInstanceId` (hint `wordpress-instance`) — picker over the operator's connected WordPress instances (credentials stripped server-side); the per-operation connector tool that used to back it was removed in the connector's open-catalog cutover (wordpress-mcp-connector#101) — content operations against the selected instance now go through the generic `wordpress_site_tools_list` / `wordpress_site_tool_call` catalog primitives instead.
- The workflow steps are: `review_publish_bundle` (approval) → `create_wordpress_draft` (agent: `@cinatra-ai/blog-wordpress-publish-agent`) → `publish_in_wordpress_admin` (manual) → `notify_publish_checkpoint_complete`. Those four step ids are the template's own and are unchanged; what the second step actually does depends on the installed version of the publish agent.
  - `@cinatra-ai/blog-wordpress-publish-agent` 0.4.x and 0.5.x (as read at 0.4.5 and 0.5.0) — the agent takes an artifact reference, never raw text and never a blog record: the post artifact and the exact representation revision the operator continued with. Once it holds the whole pinned revision, it pauses at its own `draft-confirm` HITL gate so the operator confirms BEFORE the post goes to the site, then, on a successful run, creates the page PUBLICLY through the site's own ability catalog (`wordpress_site_tools_list` to find the ability, `wordpress_site_tool_call` to invoke it — an ability whose only outcome is a draft does not satisfy the step, because a draft has no public address), and hands back the page's address, which a deterministic write-back step merges onto the post artifact's own data as `wordpressPublishedUrl`, `wordpressPublishedExternalId` and `wordpressPublishedRevisionId`. Its recipe accepts no ability that can only make a draft. The recipe has the agent publish nothing, or treat the site write as failed, when the pinned read comes back cut short, the operator declines, the site lists no ability that can publish publicly, the call fails, or the call returns a draft, pending or private page or no address; a page that is not public may then exist on the site. In each of these cases the recipe has the agent return no address: an empty write-back patch (which merges nothing), `addressWritten` false and a summary naming the cause. The publisher's own declaration records that its output contract cannot express that empty patch, and its 0.5.0 change (cinatra-ai/blog-wordpress-publish-agent#60) records no run of the publisher on a connected WordPress site, so this account rests on what the recipe asks and what the flow's graph shows, not on an observed run: the graph has an unconditional edge from the publish step to the write-back step and one from there to its end, and it has no failure node or output. Read `addressWritten` (true only when the agent built the complete three-key patch, before the write-back runs) and the summary to see whether the agent built a complete address patch, and read the post artifact's three keys to see what is stored there. It reaches no blog record and its recipe creates the page publicly, so a successful run leaves `publish_in_wordpress_admin` no draft to publish by hand. One mismatch to raise with the operator: the template hands this step `projectId`, `postId` and `wordpressInstanceId`, which are not the three inputs the agent requires (0.5.x: `postArtifactId`, `postRepresentationRevisionId`, `instanceId`; 0.4.x named the last one `wordpressInstanceId`) — tell the operator to reconcile the template with the installed agent version rather than promising this sequence reaches a published page.
  - `@cinatra-ai/blog-wordpress-publish-agent` 0.6.x (as read at 0.6.0) — the agent keeps no confirmation of its own: it has no `draft-confirm` HITL gate and its recipe pauses for nobody. Its publish step declares that it publishes, so Cinatra stops the run directly in front of that step and asks the operator on Cinatra's own publish confirmation BEFORE the step runs. Choosing Publish there runs the step; declining ends the run before the step, so nothing reaches the site and the recipe has no decline branch. The agent takes the same artifact reference and the same three inputs as 0.5.x (`postArtifactId`, `postRepresentationRevisionId`, `instanceId`). What the step does after Publish (the public page through the site's ability catalog, the write-back of the three keys, the empty patch and `addressWritten` when nothing was published, the account that rests on the recipe and the flow's graph and not on an observed run, and the mismatch with the template's inputs) is as described for 0.4.x and 0.5.x above. One difference in the result: the step runs only after the operator chose Publish, so every result it returns carries `approved: true`, a cut-short read, a missing ability or a failed call too. Read `addressWritten` and the summary, not `approved`, to see whether an address was built.
  - `@cinatra-ai/blog-wordpress-publish-agent` 0.3.x — the step did create a WordPress draft from `projectId`, `postId` and `wordpressInstanceId`, and `publish_in_wordpress_admin` was where a person published that draft by hand in the WordPress admin.
- **publish-status** shows the project's workflows + their statuses (read-only summary).

## Re-running a publish

- `@cinatra-ai/blog-wordpress-publish-agent` 0.4.x and 0.5.x (as read at 0.4.5 and 0.5.0) — there is no draft-level short-circuit, because the publish step reaches no blog record and does not use `blog_post_publish_wordpress_start`, so that primitive's `idempotentNoop` guard does not cover this path. Nothing in the step's own recipe checks whether the post is already out. On a run that reaches the site write, the confirmation the agent asks before that write is the only guard the step itself provides against a second public page: it is a decision the operator makes, not an automatic check, and the site's own ability may refuse a duplicate but nothing here says it does. So read the address stored on the post artifact (`wordpressPublishedUrl`, `wordpressPublishedExternalId`, `wordpressPublishedRevisionId`) before confirming. An address there is stored data, not a check: this step's recipe puts an address in its patch only from a page the site reports as public, so a stored address is an earlier report that the post was published at that address, though nothing here says which run or writer stored it. An empty one does not show that the post is not out, because the write-back is a separate step and the publisher's own description says it fails the write loudly on an installation without the host's write-back seam.
- `@cinatra-ai/blog-wordpress-publish-agent` 0.6.x (as read at 0.6.0) — as at 0.4.x and 0.5.x there is no draft-level short-circuit, and nothing in the step's own recipe checks whether the post is already out. The agent asks no confirmation itself: the guard against a second public page is Cinatra's publish confirmation in front of the step. It is a decision the operator makes, not an automatic check, and the site's own ability may refuse a duplicate but nothing here says it does. So read the address stored on the post artifact (`wordpressPublishedUrl`, `wordpressPublishedExternalId`, `wordpressPublishedRevisionId`) before choosing Publish there. What a stored address and an empty one show is as stated for 0.4.x and 0.5.x above.
- `@cinatra-ai/blog-wordpress-publish-agent` 0.3.x — `blog_post_publish_wordpress_start` short-circuits with an `idempotentNoop` envelope when a non-deleted WordPress draft already exists for `{projectId, postId, wordpressInstanceId}`; the existing draft refs (`wordpressDraftId`, `wordpressPostId`) are returned and no new job is enqueued. That primitive is still part of the host — the 0.4.x and 0.5.x publish step simply does not use it.

## What this skill is NOT

- It is NOT the blog AGENT (`@cinatra-ai/blog-pipeline-agent`); it is the operator workspace.
- It does NOT start/approve workflows on behalf of the user — `chat-workflow-authoring` owns proposal authoring, the dashboard owns execution.
