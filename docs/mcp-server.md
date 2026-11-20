# Local MCP server

Ported from upstream pull request [#12485](https://github.com/bambulab/BambuStudio/pull/12485)
by otarkhan, reduced and adapted for this fork.

An MCP client running on the same machine can prepare models in the open editor
window: import geometry, move and edit objects, manage plates and printer
groups, change the whitelisted process settings, slice the current plate and
export its G-code. Nothing in this build can talk to a printer or to the Bambu
cloud account.

## Enabling it

The server is **off** by default and only runs in the editor (not in the G-code
viewer).

* Preferences → General → *Local MCP server* → *Enable local MCP server*. The
  same page shows the port, the status, the address with a *Copy address*
  button, and *Copy token* / *Show token* / *Regenerate* for the access token.
* Or set `"mcp_server_enabled": true` in `BambuStudio.conf` of the data
  directory before starting the app. The port lives in `"mcp_server_port"`
  (default `27183`, allowed range 1024-65535).

## Connecting

* URL: `http://127.0.0.1:27183/mcp` — one `POST`, plain JSON responses, bound to
  the loopback interface only.
* Token: `<data_dir>/mcp.token`, created with mode `0600` on first start and
  re-secured on every read. The request needs
  `Authorization: Bearer <token>`.
* `Host` must be `127.0.0.1:<port>` or `localhost:<port>`, and `Origin`, if
  sent, must match; anything else is rejected.
* `initialize` answers protocol version `2025-11-25`. Every later request must
  send `MCP-Protocol-Version: 2025-11-25` and an `Accept` header containing
  both `application/json` and `text/event-stream`. The newer `2026-07-28`
  revision is also accepted, with the extra `Mcp-Method` / `Mcp-Name` headers
  and `_meta` fields it requires.

Standard MCP HTTP client configuration:

```json
{
  "mcpServers": {
    "bambu-studio": {
      "type": "http",
      "url": "http://127.0.0.1:27183/mcp",
      "headers": {
        "Authorization": "Bearer <contents of <data_dir>/mcp.token>",
        "MCP-Protocol-Version": "2025-11-25",
        "Accept": "application/json, text/event-stream"
      }
    }
  }
}
```

`tools/mcp_smoke.sh` in the workspace root drives the same endpoint with curl.

## Tools

72 tools, served from the manifest embedded in `McpToolsManifest.inc`
(generated from `McpToolsManifest.json`). Every listed tool has a handler.

| Area | Tools |
| --- | --- |
| App & capabilities (2) | `app_get_capabilities`, `app_get_state` |
| Printer & presets (4) | `printer_get_state`, `printer_select_nozzle`, `preset_list`, `preset_select` |
| Project (3) | `project_new`, `project_open`, `project_save` |
| Model & import (3) | `model_list`, `model_import`, `model_import_multipart` |
| Objects (10) | `object_inspect`, `object_rename`, `object_select`, `object_transform`, `object_center`, `object_drop_to_bed`, `object_mirror`, `object_delete`, `object_duplicate`, `object_reorder` |
| Selection (2) | `selection_all`, `selection_clear` |
| Plates (11) | `plate_list`, `plate_select`, `plate_create`, `plate_delete`, `plate_rename`, `plate_reorder`, `plate_lock`, `plate_move_instance`, `plate_set_print_sequence`, `plate_clone`, `plate_set_bed_type` |
| Printer groups (2) | `plate_create_for_printer`, `plate_use_another_printer` |
| Arrange, orient, history, view (6) | `arrange_start`, `orient_start`, `history_undo`, `history_redo`, `view_set`, `view_capture` |
| Settings & filament (5) | `settings_get`, `settings_effective`, `settings_update`, `settings_override`, `filament_assign` |
| Geometry & mesh (13) | `geometry_faces`, `geometry_analyze`, `object_lay_on_face`, `object_cut`, `object_split`, `object_merge`, `volume_add`, `volume_set_type`, `painting_set`, `layers_get`, `layers_set`, `mesh_repair`, `mesh_simplify` |
| Slice, preview, export (11) | `slice_get_state`, `slice_validate`, `slice_start`, `job_get`, `job_cancel`, `preview_get_summary`, `preview_get_layers`, `preview_get_toolpaths`, `export_geometry`, `export_gcode`, `export_sliced_3mf` |

Mutations run on the GUI thread and are refused with `BUSY` while a modal
dialog, a gizmo edit, a project load, a UI job or a slice is active. Slicing is
refused while the effective configuration contains a post-processing script, and
`settings_update` only accepts its whitelist of process keys.

### Printer groups

This fork can hold plates for several printers in one project. Each plate
belongs to a printer group, and the globally selected presets are always the
ones of the group of the **current** plate.

* `plate_list`, `app_get_state` and `printer_get_state` report `groupId` and the
  group's printer per plate, plus `hasPrinterGroups` and `activeGroupId`.
* `preset_list`, `preset_select`, `printer_select_nozzle`, `settings_get`,
  `settings_update`, `settings_override` and `settings_effective` name the group
  they acted on.
* `plate_create_for_printer` adds an empty plate in a group of its own;
  `plate_use_another_printer` gives an existing plate a group of its own. Both
  take an optional `printerPreset` that selects the printer of that group only,
  through the same path the sidebar uses
  (`Plater::update_objects_position_when_select_preset`), which leaves the other
  groups' plates and objects untouched. `preset_select` with `kind=printer`
  still refuses while the project holds objects, as upstream; use these two
  tools for a per-group printer instead.
* Cross-group operations are refused with a message that names the group:
  `settings_effective` and `settings_override` for a plate of another group,
  `plate_reorder` across groups, `plate_move_instance` and `plate_delete`
  with `contents=move` to a plate of another group. The beds differ, so the
  objects are not guaranteed to fit, and the presets would be the wrong ones.
* Slicing and export always work on the current plate only. There is no
  "slice all plates" tool.

## What this port leaves out, compared with the upstream PR

* **Everything that could reach a printer or the cloud.** The 20 tools in the
  upstream `McpDevices.inc` and `McpBatch.inc` (`device_*`, `ams_*`,
  `calibration_*`, `account_*`, `preferences_get` / `preferences_set` and
  `slice_start_batch`) are absent, along with their handlers, manifest entries
  and the `mcp_allow_device_actions` / `mcp_allow_account_reads` preferences
  that gated them upstream. 90 tools upstream, 70 here plus the 2 printer-group
  tools.
* **Behaviour changes for users who leave the server off.** The PR removed the
  force-cancel timeouts in `BackgroundSlicingProcess::stop()` and
  `stop_internal()`, vetoed closing the main window during a slice, made
  `ExclusiveJobGroup::start` refuse silently and `recreate_GUI` return a bool.
  None of that is ported; the fork keeps its own cancellation behaviour, and an
  orphaned worker reports its MCP run as cancelled so the tracked job still
  reaches a terminal state. The PR's `orphans` counter depended on the removed
  code and is left out as well.
* **Docs and tests.** The upstream `docs/full-mcp-parity.md`,
  `tests/mcp/legacy_tools_oracle.json`, `native_acceptance.*` and the GitHub
  workflow change are not ported. `tests/mcp/test_mcp_server.cpp` is a separate
  CMake project of its own and is not wired into this tree's test build (which
  is configured with `SLIC3R_BUILD_TESTS=OFF`), so it is left out too;
  `tools/mcp_smoke.sh` covers the endpoint end to end instead.
