**[See it live →](https://calpolyvibecoding-01.github.io/cpvc-starter/)**

# Your first website

You are about to have a real website, live on the internet, with your name on
it. You do not need to install anything and you do not need to know how to
code. Everything below happens in your browser.

**One thing to have first, it's free:** a
**[GitHub account](https://github.com)**.

> [!NOTE]
> This session moves in five parts, and each one ends at a **STOP**. When you
> hit a stop, wait there. We move as a room. If you are early, help whoever is
> next to you, that is the fastest way to get everybody through.

---

# Part 1 &middot; Get a live link

### 1. Click the green **Use this template** button

Top right of this page. Choose **Create a new repository**.

<img width="932" height="98" alt="image" src="https://github.com/user-attachments/assets/02747e67-fcc0-496a-ac82-cfd9e24af473" />

### 2. Name it and click **Create repository**

Any name works. `my-first-site` is fine. Leave everything else alone.

### 3. Turn your website on

In your new repository: **Settings** → **Pages** (left sidebar) → under
*Source* choose **Deploy from a branch** → pick **main** → **Save**.

<img width="934" height="477" alt="image" src="https://github.com/user-attachments/assets/e8d430c1-9458-49ec-8711-652a83791d8f" />

### 4. Wait about a minute, then open your site

Refresh the Settings → Pages screen. A link appears at the top that looks like:

```
https://YOUR-USERNAME.github.io/my-first-site/
```

Click it. **That is your website. It is live. Anyone in the world can open it.**

If you get a 404, wait another 30 seconds and refresh. The first build is the
slow one.

---

> [!IMPORTANT]
> ## 🛑 STOP 1 &mdash; everybody needs a live link
>
> Do not move on until your link opens and shows a page that says **It's live**.
>
> Nothing after this point works without it. If yours is not up, hand in the
> air. If yours is up, look left and right before you keep going.

---

# Part 2 &middot; Claim it

### 5. Put your name on it

In your repository, click **index.html**, then the **pencil icon** at the top
right of the file. Find `YOUR NAME HERE` near the top and change it to your
name. Scroll down, click **Commit changes**, then refresh your live site.

<img width="701" height="259" alt="image" src="https://github.com/user-attachments/assets/10c995b5-198a-433c-9a7f-acbe969d0c6d" />

You just edited a real website and shipped it. That loop, edit → commit →
refresh, is every change you will ever make today.

### 6. Submit your link to CPVC

Go to **[calpolyvibecoding.com](https://www.calpolyvibecoding.com/)**. Scroll
down until the nav bar appears, click **Login**, and create an account.

Fill in your info. Three things matter:

- **Use an email you actually check.** Everything, including Claude seat
  invites, goes to the address you sign up with.
- **Tick the box if you want a Claude seat.** You will not be considered
  without it.
- **Paste your live link.** The one from Part 1.

If it asks what you are building and you have no idea yet, put **TBD**. That is
a real answer today.

---

> [!IMPORTANT]
> ## 🛑 STOP 2 &mdash; link submitted before anyone opens Gemini
>
> This is the step people skip, and it is the one that decides who gets a
> Claude seat. Account made, seat box ticked, link pasted.
>
> Done early? Ask the person next to you if they got theirs in.

---

# Part 3 &middot; Round one, the one-liner

Open **[Gemini](https://gemini.google.com)** and sign in with a Google account.
The free version is all you need today.

Ask for a website in one sentence. Anything you want. Then paste the block
underneath it:

> make me a personal website
>
> Give me one complete HTML file. All CSS in a `<style>` tag. You may link one
> Google Font. No frameworks, no other external files, no build step. Return
> only the code.

**Then put it on your site:**

1. Copy the code with the **copy button** on the code block. Do not click and
   drag, you will grab the explanation text too.
2. Open `index.html` in your repository and click the pencil icon.
3. **Ctrl+A** (**Cmd+A** on Mac), then **Delete**. Clear the whole file.
4. Paste, commit, refresh your live site.

Deleting everything is correct. You are replacing the page, not adding to it.

It'll be okay. Not great. That's the point.

---

> [!IMPORTANT]
> ## 🛑 STOP 3 &mdash; everybody has round one live
>
> Your site should now look nothing like it did ten minutes ago.
>
> If your page looks wrecked, see the table at the bottom. It is almost always
> one thing and it takes ten seconds to fix.

---

# Part 4 &middot; Write your TARGET

Open **[`SPEC.md`](SPEC.md)** and fill in six lines:

| | |
|---|---|
| **T** &mdash; Thing | What it is, one sentence |
| **A** &mdash; Audience | Who it's for, a specific person |
| **R** &mdash; Requirements | The two or three things it must do |
| **G** &mdash; Guardrails | What it must never do |
| **E** &mdash; Experience | Sections in order, one accent color, one reference |
| **T** &mdash; Test | What "done" means |

Commit the file when you are done.

**The line people write worst is E.** It is tempting to write moods:
*"forest aesthetic with beach vibes."* Models turn moods into generic layouts.
Write structure instead: *"hero, story, three highlights, contact. One accent
color, deep green. Full-height hero, lots of whitespace. Should look like a
clean personal site, not a resume."* Same line, completely different result.

---

> [!IMPORTANT]
> ## 🛑 STOP 4 &mdash; six lines written before anyone prompts again
>
> This is the actual lesson of the session. Everything else today is
> logistics.
>
> Do not skip to Part 5 with a half-filled TARGET. The comparison only works
> if you wrote it down properly.

---

# Part 5 &middot; Round two, the same tool with a better input

Back to Gemini:

> Build me a website from this spec:
> [paste your six TARGET lines]
>
> Give me one complete HTML file. All CSS in a `<style>` tag. You may link one
> Google Font. No frameworks, no other external files, no build step. Return
> only the code.

Copy, select all in `index.html`, delete, paste, commit, refresh.

**Now open round one and round two side by side. That comparison is the entire
lesson.** Same tool, same model, same person, twenty minutes apart. The only
thing that changed is that you wrote it down first.

---

> [!IMPORTANT]
> ## 🛑 STOP 5 &mdash; last one
>
> Everybody has a live site built from their own TARGET. That is the session.
>
> Anything past this point is yours to keep going with during open build time.

---

## Keep going: round three

The round people skip. Ask for one specific change:

> make the hero full height, tighten the spacing, and use one accent color
> instead of three

Copy, paste, commit, refresh. Iterating isn't cheating, it's the last step of
The Loop, and it's where output stops looking generated.

Then add it to your phone: open your link in Safari or Chrome, hit **Share** →
**Add to Home Screen**. It opens like an app.

---

## If something looks broken

| What you see | What it is |
|---|---|
| **404 page not found** | The first build takes about a minute. Wait and refresh. If it persists, check the file is named exactly `index.html`, all lowercase. |
| **Site did not change** | It caches for a minute. Hard refresh: `Ctrl+Shift+R` on Windows, `Cmd+Shift+R` on Mac. |
| **The page looks wrecked** | Gemini gave you a fragment, not a whole page. What you paste should start with `<!DOCTYPE html>`. Go back to Gemini and ask for "one complete HTML file, return only the code." To undo: open the file, click **History**, open the version before your change, copy it back. |
| **Half the page is Gemini's explanation** | You click-and-dragged instead of using the copy button. Clear the file and paste again. |
| **Cannot find the pencil icon** | Click the file name first, *then* look at the top right of the file box. |
| **No laptop** | Pair up with whoever is next to you. Put both names in the footer and both of you ship it. |
