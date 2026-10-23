# Site editor setup: CMS, Cloudflare login worker, GitHub

First Created On: 23 Oct 2026
Last Modified On: 23 Oct 2026

## Verdict

Editors change the site at https://mahidesai.nz/admin. Every save becomes a pull request on GitHub. Nothing goes live until the owner merges it. This file explains each part, how it was set up, and how to run and fix it.

## 1. How the parts fit together

| Part | What it does | Where it lives |
|---|---|---|
| Static site | The public website | GitHub repo desank/mahidesai-website, served by GitHub Pages |
| Sveltia CMS | The editing screen at /admin | The admin folder in the repo (version pinned in admin/config.yml) |
| Login worker | Finishes the "Sign in with GitHub" step for the CMS | Cloudflare Worker named sveltia-cms-auth |
| GitHub OAuth app | Lets the worker ask GitHub who the editor is | GitHub, Settings, Developer settings, OAuth Apps, "Mahi site editor" |
| Branch rule on main | Blocks direct pushes to main | Repo Settings in GitHub |
| Collaborator role | Decides what the editor may do | Repo Settings, Collaborators |

Why a worker is needed: GitHub will not hand a login token to a static site directly. The small worker holds a secret, does the exchange, and passes the token back to the CMS.

Login flow in plain steps:
1. The editor opens /admin and taps Sign in with GitHub.
2. The CMS sends the browser to the worker (the base_url in admin/config.yml).
3. The worker sends the browser to GitHub.
4. GitHub asks the editor to approve the app, then returns to the worker at /callback.
5. The worker gives the login token to the CMS. The editor is in.

## 2. Accounts you need

- GitHub account of the owner (owns the repo).
- GitHub account of the editor (free account is fine).
- Cloudflare account of the owner (free plan is fine).

Tip for phones: the GitHub and Cloudflare settings pages are easier on a laptop. On a phone, open the page in the browser, tap the three dots, then Request Desktop Site.

## 3. Setup steps, in the order they were done

### Step A. Repo and branch protection

Goal: nobody can push straight to main, so every change is reviewed.

1. Open github.com/desank/mahidesai-website.
2. Tap Settings in the tab row (swipe the row sideways if you do not see it).
3. Open Branches (some accounts show Rules, then Rulesets instead).
4. Add a rule for the branch main.
5. Turn on "Require a pull request before merging".
6. Leave a bypass for the owner so you can merge your own pull requests. Admins can bypass, which is why the editor must not be an Admin.
7. Save.

Check it works: try a direct push or a direct edit to main. It should be blocked and offer a pull request instead. This was tested, and the push was blocked.

### Step B. Deploy the login worker on Cloudflare

Goal: have a small private login service at an address like https://sveltia-cms-auth.desank.workers.dev.

1. Sign up free at dash.cloudflare.com/sign-up and sign in.
2. Open github.com/sveltia/sveltia-cms-auth in a browser and tap the Deploy to Cloudflare Workers button. This opens Cloudflare's Create an app screen.
3. On the Git account section, tap New GitHub connection.
4. On the GitHub screen, approve Cloudflare. If GitHub asks which repositories, choose All repositories (Step I narrows this afterwards). Choosing "Only select repositories" with nothing ticked may block the deploy.
5. Cloudflare may then show a "Select a repository" list of your own repos. Do not pick any of them. Scroll down in the box and tap "Clone a public repository via Git URL".
6. Paste https://github.com/sveltia/sveltia-cms-auth and tap Next.
7. On the final screen check: Git account is your GitHub username, "Create private Git repository" is ticked, project name is sveltia-cms-auth.
8. Tap Deploy.
9. When it finishes, the worker Overview page shows "No URLs enabled". That is expected. Go to the next step.

### Step C. Switch on the workers.dev address

1. In Cloudflare, open Workers & Pages, then the sveltia-cms-auth worker.
2. Tap the Domains tab in the tab row (between Issues and Access). It is not inside Settings.
3. On the workers.dev line, tap Enable and confirm.
4. Go back to Overview. It now shows the address, which looks like sveltia-cms-auth.desank.workers.dev.
5. Write down the full address with https:// in front. This is the worker URL used below.

### Step D. Create the GitHub OAuth app

Goal: let the worker ask GitHub who the editor is.

1. Go to github.com/settings/applications/new (or Settings, Developer settings, OAuth Apps, New OAuth App).
2. Application name: Mahi site editor
3. Homepage URL: https://mahidesai.nz
4. Authorization callback URL: the worker URL plus /callback, for example https://sveltia-cms-auth.desank.workers.dev/callback
5. Tap Register application.
6. On the next page, copy the Client ID.
7. Tap Generate a new client secret and copy the secret straight away. GitHub shows it only once.

Never paste the client secret into a chat, an email, the repo or this file. If it leaks, go back to this page, tap Generate a new client secret, then repeat Step E with the new value.

### Step E. Give the worker its keys

1. In Cloudflare, open the worker and tap the Settings tab.
2. Find Variables and secrets (also called Runtime variables and secrets).
3. Tap Add variable three times and enter:

| Name | Type | Value |
|---|---|---|
| GITHUB_CLIENT_ID | Text | The Client ID from Step D |
| GITHUB_CLIENT_SECRET | Secret | The client secret from Step D |
| ALLOWED_DOMAINS | Text | mahidesai.nz |

4. Tap Deploy (or Save and deploy).
5. An orange banner about updating the Wrangler configuration may appear. It is only a reminder. You can close it.
6. Scroll down and check all three names are listed. The secret value shows as hidden.

ALLOWED_DOMAINS limits which sites may use this worker. If it is wrong, sign-in fails after the GitHub step.

### Step F. Point the CMS at the worker

Goal: tell the CMS where the login worker is. This is one line in admin/config.yml.

1. On github.com, open the repo and go to admin/config.yml.
2. Tap the pencil icon to edit.
3. In the backend section, make sure these lines are present and lined up with each other:

```yaml
backend:
  name: github
  repo: desank/mahidesai-website
  branch: main
  base_url: https://sveltia-cms-auth.desank.workers.dev
```

4. Replace desank with the real worker address from Step C.
5. Commit to a new branch and tap Propose changes, then Create pull request.
6. As the owner, merge the pull request using your bypass.
7. Wait a few minutes for GitHub Pages to rebuild.

Also check these lines in admin/config.yml:
- repo must match the real repo name exactly (desank/mahidesai-website).
- branch is main.
- publish_mode is editorial_workflow. This is what makes each save a pull request.

### Step G. Test the owner login

1. Open https://mahidesai.nz/admin.
2. Tap Sign in with GitHub.
3. Approve the "Mahi site editor" app when GitHub asks.
4. You should land inside the editor. This was tested and it worked.

### Step H. Add the editor

Goal: the editor can write content, but cannot publish.

1. Ask the editor for their GitHub username. If they have no account, they create one free at github.com/signup.
2. In the repo, open Settings, then Collaborators.
3. Tap Add people.
4. Type the editor's GitHub username and choose them from the list.
5. Choose the Write role. Do not choose Admin or Maintain. Admin can bypass the branch rule and publish without review.
6. Send the invite. The editor gets an email and a notification on GitHub. They must accept it. The invite expires after 7 days. If it lapses, send it again.
7. After they accept, ask them to open https://mahidesai.nz/admin and tap Sign in with GitHub.
8. GitHub asks them to approve "Mahi site editor". They tap Authorize.
9. They make a small change, tap Save, and the CMS opens a pull request in the repo.
10. You get a GitHub notification. Open the pull request, check the Files changed tab, and tap Merge pull request.
11. The site updates after GitHub Pages rebuilds, usually within a few minutes.

To remove the editor later:
1. Repo Settings, Collaborators, tap the remove icon next to their name.
2. Optional: ask them to revoke the app at github.com/settings/applications.

### Step I. Narrow Cloudflare's access to GitHub

Cloudflare needed GitHub access once, to create its private copy of the worker code. After the deploy, narrow it.

1. In GitHub, open Settings, then Applications, then Installed GitHub Apps.
2. Tap Configure next to Cloudflare Workers and Pages.
3. Under Repository access, choose Only select repositories.
4. Pick sveltia-cms-auth only, then Save.
5. Test the editor login again at /admin to confirm nothing broke.

To remove Cloudflare's access fully, tap Uninstall instead. The worker keeps running. It will just stop redeploying when its own repo changes, which you do not need.

Do not touch the OAuth app called Mahi site editor under Developer settings. That app runs the editor login.

## 4. Day-to-day use

Editor:
1. Open https://mahidesai.nz/admin and sign in with GitHub.
2. Pick a section, edit the entry, tap Save.
3. The change waits as a pull request. It is not live yet.
4. Tell the owner it is ready, or wait for the merge.

Owner:
1. Open the repo and tap the Pull requests tab.
2. Open the request and read the Files changed tab.
3. Tap Merge pull request, then Confirm merge.
4. Wait a few minutes. Reload the site.

## 5. Maintenance

| Task | What to do |
|---|---|
| Change the editor | Settings, Collaborators: remove the old name, add the new one with the Write role |
| Client secret leaked | GitHub OAuth app: Generate a new client secret, then update GITHUB_CLIENT_SECRET in Cloudflare and Deploy. Delete the old secret in GitHub |
| Worker address changed | Update the callback URL in the OAuth app and base_url in admin/config.yml (via pull request) |
| Allowed site changed | Update ALLOWED_DOMAINS in Cloudflare and Deploy |
| Upgrade the CMS | Change the pinned version in the admin folder through a pull request, then test the login |
| Turn editing off | Remove the editor under Collaborators. For a full stop, disable the workers.dev address in the worker's Domains tab |

## 6. Security notes

- Editors have the Write role. Branch protection on main means they cannot publish.
- The client secret lives only in Cloudflare as a Secret. It is not in the repo.
- The worker's private repo copy contains public code only, no secrets.
- Cloudflare's access to GitHub is narrowed to one repo (Step I).
- Review each pull request before merging. Check that it only touches the content files the editor should change.
- Keep the owner account's two-factor sign-in on for GitHub and Cloudflare.

## 7. Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| Sign-in page shows an error straight away | The worker address is wrong or workers.dev is disabled. Check the Domains tab in Cloudflare and base_url in admin/config.yml |
| GitHub says "redirect_uri mismatch" | The callback URL in the OAuth app must be the worker URL plus /callback, with no extra slash or spaces |
| Login finishes but nothing happens, or a blank page | Check ALLOWED_DOMAINS is mahidesai.nz, and that the client ID and secret match the current OAuth app |
| Editor signs in but cannot see content | Check the repo line in admin/config.yml, check the editor accepted the invite, and check they have the Write role |
| Editor cannot save | The Write role is missing, or the invite is still pending |
| Pull request created but site not updated | The pull request is not merged yet, or GitHub Pages is still rebuilding. Check the Actions tab |
| Changes land straight in main | The editor has Admin, or the branch rule is off. Fix the role or the rule |
| Cloudflare deploy asks for a repository | Choose "Clone a public repository via Git URL" and paste https://github.com/sveltia/sveltia-cms-auth |
| Deploy fails with a permission error | Give Cloudflare access to All repositories, deploy, then narrow it again (Step I) |

## 8. Checklist

- [ ] Branch rule on main in place and tested
- [ ] Worker deployed and workers.dev enabled
- [ ] OAuth app created with the right callback URL
- [ ] Three variables saved in Cloudflare
- [ ] base_url set in admin/config.yml and merged
- [ ] Owner login tested at /admin
- [ ] Editor invited with the Write role and invite accepted
- [ ] Editor test save created a pull request, and the owner merged it
- [ ] Cloudflare access to GitHub narrowed or removed

## 9. Version history

| Version | Date | Change |
|---|---|---|
| 1 | 23 Oct 2026 | First version |
