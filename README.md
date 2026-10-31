# FormFlow — MVP Starter

This is a runnable, single-file prototype based on the FormFlow README.

## Run it
1. Open `index.html` in a modern browser, or publish it using GitHub Pages.
2. Add and edit fields in the left and middle panels.
3. Submit the customer preview to test the local submission inbox.
4. Use **Export form** to download the form definition as JSON.
5. Use **Export CSV** to download the submissions currently stored in this browser.

## Current scope
Included:
- Editable form title, description, and submit label
- Add common field types
- Edit labels, placeholders, required status, and choice options
- Reorder and delete fields
- Customer preview with browser validation
- Local draft and demo submission storage using `localStorage`
- Export form JSON and submissions CSV

Not included yet:
- Accounts, organization/workspace isolation, team invites
- Cloud database or cross-device storage
- Real public share links / embedded forms
- Secure document uploads or virus scanning
- Email notifications
- Backend authorization, audit logs, and production security

**Do not collect real or sensitive customer data in this prototype.** Browser localStorage is not a secure production database. For the full platform described in the original README, connect a backend/database and secure file storage before launch.

## Publish with GitHub Pages
1. Upload `index.html` to the root of the `FORM-BUILDER` repository.
2. In GitHub, open **Settings → Pages**.
3. Under Build and deployment, choose **Deploy from a branch**.
4. Choose branch `main` and folder `/ (root)`, then Save.
5. Wait for the published URL to appear in the Pages section.

Repository: https://github.com/ellysemo-ux/FORM-BUILDER
