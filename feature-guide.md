---
layout: default
title: So you want to implement a feature?
---

When you want to implement a new significant feature in the compiler, you need to go through this process to make sure everything goes smoothly:

## The @rfcbot (p)FCP process

If you are making an uncontroversial change, then you can get by with just writing a PR and getting r+ from someone who knows that part of the code. However, if the change is potentially controversial, it would be a bad idea to push it without consensus from the rest of the team (both in the "distributed system" sense to make sure you don't break anything you don't know about, and in the social sense to avoid PR fights).

If such a change seems to small to require a full formal RFC process (e.g. a big refactoring of the code, or a "technically-breaking" change, or a "big bugfix" that basically amounts to a small feature) but is still too controversial or big to get by with a single r+, you can start a pFCP (or, if you don't have r+ rights, ask someone who has them to start one - and unless they have a concern themselves, they should).

Again, the pFCP process is only needed if you need consensus - if you don't think anyone would have a problem with your change, it's ok to get by with only an r+. For example, it is OK to add or modify unstable command-line flags or attributes without an pFCP for compiler development or standard library use, as long as you don't expect them to be in wide use in the nightly ecosystem.

We're trying to find a better process for iterating on features on nightly, but if you are trying to write a feature that is not too bikesheddy, you could try to ask for an pFCP to land it on nightly under a feature gate. That might work out. However, some team-member might still tell you to go write an actual RFC (however, someone still needs to write an RFC before that feature is stabilized).

You don't have to have the implementation fully ready for r+ to ask for a pFCP, but it is generally a good idea to have at least a proof of concept so that people can see what you are talking about.

That starts a "proposed final comment period" (pFCP), which requires all members of the team to sign off the FCP. After they all do so, there's a week long "final comment period" where everybody can comment, and if no new concerns are raised, the PR/issue gets FCP approval.

After an PR/issue had got FCP approval, team-members should help along attempts to implement it, and not block r+ on the controversy that prompted the FCP. If significant new issues arise after FCP approval, then it is ok to re-block the PR until these are resolved.

Teams and team members: that means that even if someone is trying to paint the bikeshed your least favorite color, you should help them along. Don't be a slow reviewer.

### A few etiquette notes about (p)FCPs

FCPs are a place where people unfamiliar with the Rust teams and process interact with the design process. This means that it is extra important for communication to be clear there. Don't assume the PR's writer knows when the next lang team meeting is.

pFCP checkbox ticking is not supposed to be a place where PRs stall due to low priority - it is supposed to quickly either reach consensus or find a concern. If there is no real obstacle to consensus on the issue, the relevant team should sign off the PFCP reasonably quickly (a week or two). 

Teams: please review pFCP items during the team meetings to make sure this happens. If you find a real concern, please post a summary on the issue, even if you already discussed it on the team IRC channel, or worse, at a late-hours in-person meeting.

If you think a pFCP item requires further design work which you are too busy to do right now, please say so on the issue/PR thread. This might appear rude, but it's less rude than leaving a PR accumulating bitrot for 2 months because you have no idea what to do with it.

### How to technically run a pFCP

pFCPs are run using the @rfcbot bot. For more details about how to correctly use rfcbot, see [the rfcbot guide], but the gist of it is that someone with privileges first tags your issue with a list of teams, and then someone writes this command in a comment in the issue thread:  
```
@rfcbot fcp merge
```
or  
```
@rfcbot fcp close
```

That will cause rfcbot to comment its ticky-box comment.

[the rfcbot guide]: https://github.com/dikaiosune/rfcbot-rs/blob/master/RFCBOT.md

## The logistics of writing features 

There are a few "logistic" hoops you might need to go through in order to implement a feature in a working way.

The more boring details are listed in the Rust repository's [CONTRIBUTING.md], so I'll just link to it here. You should try to at least skim it fully - it contains a bunch of nice tips.

[CONTRIBUTING.md]: https://github.com/rust-lang/rust/blob/master/CONTRIBUTING.md

### Warning Cycles

In some cases, a feature or bugfix might break some existing programs in some edge cases. In that case, you might want to do a crater run to assess the impact and possibly add a warning cycle, following the [rustc bug-fix procedure](rustc-bug-fix-procedure.html).

### Stability

We [value the stability of Rust]. Code that works and runs on stable should (mostly) not break. Because of that, we don't want to release a feature to the world with only team consensus and code review - we want to gain real-world experience on using that feature on nightly, and we might want to change the feature based on that experience.

To allow for that, we must make sure users don't accidentally depend on that new feature - otherwise, especially if experimentation takes time or is delayed and the feature takes the trains to stable, it would end up *de facto* stable and we'll not be able to make changes in it without breaking people's code.

The way we do that is that we make sure all new features are *feature gated* - they can't be used without a enabling a *feature gate* (`#[feature(foo)]`), which can't be done in a stable/beta compiler. See the [stability in code](#stability-in-code) section for the technical details.

Eventually, after we gain enough experience using the feature, make the necessary changes, and are satisfied, we expose it to the world using the stabilization process described [here](stabilization-guide.html). Until then, the feature is not set in stone: every part of the feature can be changed, or the feature might be completely rewritten or removed. Features are not supposed to gain tenure by being unstable and unchanged for a year.

[value the stability of Rust]: https://github.com/rust-lang/rfcs/blob/master/text/1122-language-semver.md

### Tracking Issues

To keep track of the status of an unstable feature, the experience we get while using it on nightly, and of the concerns that block its stabilization, every feature-gate needs a tracking issue.

General discussions about the feature should be done on the tracking issue.

For features that have an RFC, you should use the RFC's tracking issue for the feature.

For other features, you'll have to make a tracking issue for that feature. The issue title should be "Tracking issue for YOUR FEATURE" and it should have the `B-unstable` & `C-tracking-issue` tags, along with the tag for your subteam (e.g. `T-compiler` if this is a compiler feature).

For tracking issues for features (as opposed to future-compat warnings), I don't think the description has to contain anything specific. Generally we put the list of items required for stabilization using a github list, e.g.

```
**Steps:**

- [ ] Implement the RFC (cc @rust-lang/compiler -- can anyone write up mentoring instructions?)
- [ ] Adjust documentation ([see instructions on forge][doc-guide])
- Note: no stabilization step here.
```

## Stability in code

In order to implement a new unstable feature, you need to do the following steps:

1. Open a [tracking issue](#tracking-issues) - if you have an RFC, you can use the tracking issue for the RFC.
2. Pick a name for the feature gate (for RFCs, use the name in the RFC).
3. Add a feature gate declaration to `libsyntax/feature_gate.rs`
    In the active `declare_features` block:  
    ```
    // description of feature
    (active, $feature_name, "$current_nightly_version", Some($tracking_issue_number))
    ```
    
    For example  
    ```
    // allow '|' at beginning of match arms (RFC 1925)
    (active, match_beginning_vert, "1.21.0", Some(44101)),
    ```
    
    The current version is not actually important - the important version is when you are *stabilizing* a feature.
4. Prevent usage of the new feature unless the feature gate is set.  
    You can check it in most places in the compiler using the expression  
    ```
    tcx.sess.features.borrow().$feature_name
    ```
    
    If the feature gate is not set, you should either maintain the pre-feature behavior or raise an error, depending on what makes sense.
5. Add a test that the feature can't be used without a feature gate, under `src/test/compile-fail/feature-gate-$feature_name.rs`.
6. Add a section to the unstable book, in `src/doc/unstable-book/src/language-features/$feature_name.md`.
7. Write a lots of tests for the new feature. PRs without tests will not be accepted!
8. Get your PR reviewed and land it. You have now successfully implemented a feature in Rust!
