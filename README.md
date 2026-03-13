# micr.dev

This repo now owns the root `https://micr.dev` site and the two path-based pages that stay on the main domain:

- `https://micr.dev/`
- `https://micr.dev/tree/`
- `https://micr.dev/wip/`

The other projects were split into their own repos and domains:

- `https://about.micr.dev/`
- `https://quarzite.micr.dev/`
- `https://microkeebs.micr.dev/`
- `https://anonq.micr.dev/`

## Contents

- Root 3D landing page and Spline assets
- `tree/` link hub
- `wip/` placeholder page
- Root redirects, headers, sitemap, and well-known files

## Deployment Notes

- `netlify.toml` in this repo is the post-migration cutover config
- Do not deploy this repo to production until the subdomain sites are already live
- Old path URLs are expected to `301` to the new subdomains once cutover happens

## License

**© micr.dev 2025 ∷ all rights reserved.**

All code, design, writing, and media assets in this repository are fully owned by me.
Nothing in this repo may be copied, reused, modified, or distributed without explicit written permission.
