# Veronica Mumm's Music — Project Plan

## Goal
Create a static website for "Veronica Mumm's Music" (in-home violin, cello, and viola lessons in Richmond, VA) and deploy it to a Google Cloud Storage bucket using free/near-free services only.

## Progress

### Initial site scaffold (2026-05-03)
Built the complete single-page static site with vanilla HTML+CSS (minimal JS). Chose a CSS-first approach with no external dependencies — inline SVG social icons, CSS-only hamburger menu, system font stack. Created a GCP deployment guide documenting the bucket hosting setup, DNS configuration, HTTPS options (Cloudflare free tier), and future ownership transfer process. The site is ready for local preview and GCS deployment.

## Next Steps
- Set up a remote repository
- Follow `docs/gcp-setup.md` to deploy to GCS
- Replace placeholder contact info with real details
- Add real social media URLs
- Configure HTTPS (Cloudflare or Firebase)
