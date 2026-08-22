# HubSpot CRM Playbook

## Quick Reference

| Goal                  | Tool                        | Notes                                                                      |
| --------------------- | --------------------------- | -------------------------------------------------------------------------- |
| Create a company      | `hubspot_create_company`    | Use `website_url` when you want HubSpot to infer the domain.              |
| Create a contact      | `hubspot_create_contact`    | Prefer `email` for stable identity matching.                               |
| Create a deal         | `hubspot_create_deal`       | Use `deal_stage` and `deal_probability` only when you know the pipeline.  |
| Create a note         | `hubspot_create_note`       | `time_stamp` is required. Add associations to place it on a record timeline. |
| Create a task         | `hubspot_create_task`       | `task_type` should usually be `TODO`.                                     |
| Update a record       | `hubspot_update_*`          | Always include `id` and only the fields you want to change.               |
| Delete a record       | `hubspot_delete_*`          | Hard delete only when the target should disappear from HubSpot.           |
| Browse records        | `hubspot_list_*`            | Use for paging and record inspection.                                     |
| Fetch one record      | `hubspot_get_object`        | Best when you already have the record ID.                                 |
| Search records        | `hubspot_search_objects`    | Best for fuzzy lookups and filters.                                       |

## Practical Notes

- HubSpot normalizes most writes to CRM property names such as `firstname`, `lastname`, `hubspot_owner_id`, and `dealstage`.
- For object-heavy workflows, prefer `hubspot_search_objects` over broad listing when you need filters or quick lookup by email/domain.
- The `hubspot_list_objects` and `hubspot_get_object` helpers work for standard objects and custom objects when you know the object type.
- When using notes or tasks, add associations up front so the activity lands on the right record timeline.
- For large syncs, use batch operation tools (e.g., `hubspot_batch_create_contacts`) rather than looping single-record calls.
- Discover schema, pipeline stages, and owner IDs using `hubspot_get_pipelines` and `hubspot_get_owners` before writing — never guess enum values.

## Common Workflows

### Upsert a contact

1. Call `hubspot_search_objects` with `filterGroups` by email to check for an existing record.
2. If found, call `hubspot_update_contact` with the record ID.
3. If not found, call `hubspot_create_contact`.

### Sync a list of enriched contacts

1. Use batch create/update tools for the contact objects.
2. Associate each contact with the relevant company using `hubspot_create_association`.
3. Verify the final record count with `hubspot_list_objects` or a search call.

### Log outreach activity

1. Call `hubspot_create_note` with the contact ID in `associations`.
2. Set `time_stamp` to the send time (ISO 8601).
3. Include the message body or summary in `hs_note_body`.

---

## Save to Context Lake

Save CRM operation results for auditability and to avoid duplicate writes in future sessions:

```json
POST /context/
{
  "serverId": "26",
  "content": "HubSpot operation complete. Action: {{created/updated/searched — contacts/deals/companies}}. Records affected: {{count, IDs or names}}. Notes or tasks created: {{count}}. Any errors: {{list}}.",
  "messageId": "msg-{{Date.now()}}",
  "userId": "claude-user",
  "username": "Claude",
  "createdDate": "<new Date().toISOString() — exact UTC timestamp, never local time or an approximation>",
  "channelId": "claude-session",
  "threadId": "<cowork-session-id>"
}
```
