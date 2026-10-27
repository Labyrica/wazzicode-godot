# Godot Vibe OS repository

This is a pnpm TypeScript workspace plus a Godot 4 editor addon and a Tauri desktop app.

- Never guess a Godot API. Verify unfamiliar signatures in the installed 4.7 source/docs or through `godot_reflect`.
- Do not hand-edit `.tscn` or `.tres` when a dedicated editor RPC fits the change.
- After addon changes run the real fixture integration in `tests/godot/integration.mjs`.
- After TypeScript changes run the affected package typecheck and tests; before completion run the full gates in README.
- Do not call import or `--check-only` a unit test. Project tests are `not_configured` unless a real runner is invoked.
- Preserve localhost-only binding, per-launch token auth, project identity validation, request bounds, UndoRedo scene mutations, SHA-guarded file edits, snapshots, and action logging.

Use `node apps/cli/bin/gvibe <command>` for the local CLI.

## Releasing

Foundry for Godot ships through the Labyrica release server (https://labyrica.com/admin), not GitHub Releases or GitHub Actions. Employees install and update it with Labyrica Dispatch after signing in. App id: `foundry-godot`. Signing key: `~/.tauri/foundry-godot.key` (empty password); its public half is the app's `pubkey` in `labyrica-dispatch/catalog/apps.json`.

1. Bump the version in `apps/desktop/package.json`, `apps/desktop/src-tauri/tauri.conf.json` and `apps/desktop/src-tauri/Cargo.toml`; tag `desktop-v<version>`.
2. Build on this Mac:
   ```sh
   pnpm install --frozen-lockfile && pnpm build
   cd apps/desktop
   node scripts/bundle-gvibe.mjs && node scripts/fetch-whisper.mjs
   for t in aarch64-apple-darwin x86_64-apple-darwin x86_64-pc-windows-msvc; do node scripts/fetch-node-sidecar.mjs --target $t; done
   pnpm exec tauri build --config src-tauri/tauri.bundle.conf.json --target aarch64-apple-darwin --bundles app
   pnpm exec tauri build --config src-tauri/tauri.bundle.conf.json --target x86_64-apple-darwin --bundles app
   pnpm exec tauri build --config src-tauri/tauri.bundle.conf.json --runner cargo-xwin --target x86_64-pc-windows-msvc --bundles nsis
   ```
3. The app has no updater plugin, so Tauri makes no `.app.tar.gz` and signs nothing: package and sign by hand. No spaces in uploaded names.
   ```sh
   COPYFILE_DISABLE=1 tar -C src-tauri/target/aarch64-apple-darwin/release/bundle/macos -czf Foundry.for.Godot_aarch64.app.tar.gz "Foundry for Godot.app"
   COPYFILE_DISABLE=1 tar -C src-tauri/target/x86_64-apple-darwin/release/bundle/macos -czf Foundry.for.Godot_x64.app.tar.gz "Foundry for Godot.app"
   cp "src-tauri/target/x86_64-pc-windows-msvc/release/bundle/nsis/Foundry for Godot_<version>_x64-setup.exe" Foundry.for.Godot_<version>_x64-setup.exe
   export TAURI_SIGNING_PRIVATE_KEY="$(cat ~/.tauri/foundry-godot.key)" TAURI_SIGNING_PRIVATE_KEY_PASSWORD=""
   for f in Foundry.for.Godot_*; do npx @tauri-apps/cli@2 signer sign "$f"; done
   ```
4. Upload and publish to the release server:
   ```sh
   node ~/Repos/labyrica-hub/bin/push.ts login roberto-admin   # once; asks for the authenticator code
   node ~/Repos/labyrica-hub/bin/push.ts publish foundry-godot <version> <files...>
   ```
   Every file needs its signature next to it (`<file>.sig`). The server rejects unsigned files and files signed with any key other than this app's.
5. Delete the build cache afterwards (`cargo clean` in `apps/desktop/src-tauri`, and remove the packaged files).

If the key is lost, generate a new one (`npx @tauri-apps/cli@2 signer generate --ci -p "" -w ~/.tauri/foundry-godot.key`), then update the `pubkey` in `labyrica-dispatch/catalog/apps.json` and the app's `pubkey` row in the release server database (`/data/hub.db` in the `labyrica-hub` pod; the dashboard has no control for it yet). Builds signed with the old key stop installing.
