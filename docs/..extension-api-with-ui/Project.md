# Project Diary

Aimed at extending Zed's Extensions API to support additions to UI.

Using:
- Tools:
  - Zed
  - Claude
- Repositories
  - zed
  - mentenaz.Forge.Super

Aware of https://github.com/zed-industries/embedded_gpui and independent approach purposeful.

## Initial planning round.

Cloned into same folder `zed`, `metenaz.Forge.Super` and personal zed fork named `extension-api-with-ui`, created `extending-zed-api` folder, all four folders added to project.

Created this and another four `.md` files in the later folder, copied `.zed` from `zed` for later debugging.

Initialized:

> Opus 5.5 Medium
```md
Zed Extension API adding UI support.

With Sonnet agents:
1. Read https://github.com/zed-industries/zed/discussions/53403 write points to [@thread.md](file:///home/tiago/Repositories/extending-zed-api/thread.md)
2. With 1. Read [@zed](file:///home/tiago/Repositories/zed/) Write [@API.md](file:///home/tiago/Repositories/extending-zed-api/API.md) all found regarding Zed's extension API, and how would expand.
3. With 1. Read [@mentenaz.Forge.Super](file:///home/tiago/Repositories/mentenaz.Forge.Super/) Write [@applications.md](file:///home/tiago/Repositories/extending-zed-api/applications.md) all features added onto zed.

With Opus agent:
4. With 2. 3. Compare [@API.md](file:///home/tiago/Repositories/extending-zed-api/API.md) and [@applications.md](file:///home/tiago/Repositories/extending-zed-api/applications.md) Write [@plan1.md](file:///home/tiago/Repositories/extending-zed-api/plan1.md) how would implement features as extensions based on API.

Rules for the API:
1. Must use GPUI, rendered to WASM, integrating into Zed's existing structure and pipeline.
2. Extensions may add UI only in declarative form, JSON. API will process UI-declaration into elements.
3. Extensions must declare permissions. Make permission model filesystem, network, processe. Maximum detail.
```

As gh was not installed, followed:

> Opus 5.5 Medium
```md
1. Complete thread.md
2. Update applications.md if needed
3. Update plan1.md if needed
```

As token limit ran out:

> Sonnet 5.5 Medium
```md
Continue
```

## Sharing inital planning

Decided to add this into extension-api-with-ui docs, into `..extension-api-with-ui` named so to remind me/others that this folder is a planning folder.

> Sonnet 5.5 Medium
```md
Move [@extending-zed-api](file:///home/tiago/Repositories/extending-zed-api/) into [@..extension-api-with-ui](file:///home/tiago/Repositories/extension-api-with-ui/docs/..extension-api-with-ui/)
```
