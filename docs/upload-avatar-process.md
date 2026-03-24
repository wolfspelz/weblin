# Avatar Items: Upload Avatar & Custom Avatar

Documentation of the avatar upload system - both the original Upload Avatar (single animations) and the newer Custom Avatar (ZIP with config.xml). Covers the end-to-end process from item creation to rendered animation on the web page.

## Overview

There are two avatar item types that share a common base class:

- **Upload Avatar** - Upload individual animation files one at a time. The server generates config.xml automatically. Supports one animation per type.
- **Custom Avatar** - Upload a ZIP containing a pre-built config.xml and all animation files. Supports multiple animations per type with custom probabilities.

Both items use the same client-side rendering pipeline (Avatar.ts, AnimationsXml.ts, Animation Proxy).

## Phase 1: Item Definition

**File:** `Items/n3q.Items/Templates/Standard.cs` (line ~767)

The `UploadAvatar` template defines:
- **Aspects:** `AvatarAspect`, `ActivatableAspect`, `IframeAspect`
- **IframeAspect:** hidden, owner-only popup (opens the upload UI)
- **16 animation types:** icon, idle, moveleft, moveright, chat, wave, kiss, yawn, dance, cheer, laugh, deny, agree, angry, cry, clap
- **Key properties:**
  - `Pid.AvatarAnimationsUrl` - URL to generated config.xml
  - `Pid.ActivatableIsActive` - active/inactive toggle
  - `Pid.IsUnrezzedAction` - true (action item)

## Phase 2: Opening the ItemFrame (Client -> Server)

### Client Side
1. User clicks the item in their backpack
2. Extension opens an `ItemFramePopup` (contentscript/ItemFramePopup.ts)
3. `ItemFrameContextFactory.ts` generates a signed context token containing:
   - userId, itemId, roomId, inventoryId, providerId, entropy
   - HMAC-SHA256 signature using the user's access token
4. iframe URL: `https://webit.vulcan.weblin.com/ItemFrame/UploadAvatar?context={base64Token}`

### Server Side
5. `ItemFrameModel.cs` validates the signature against stored access tokens
6. Loads item properties from Orleans InventoryGrain
7. Renders the upload UI (`UploadAvatar.cshtml`)

### Communication Protocol
- iframe <-> extension via `window.postMessage()` with magic headers
- Message types defined in `WeblinClientIframeApi.ts`
- Key messages: `Item.SetProperty`, `Window.Position`, `Window.Close`

## Phase 3: Uploading Animations

**Files:**
- `Items/n3q.WebIt/Pages/ItemFrame/UploadAvatar.cshtml` (frontend)
- `Items/n3q.WebIt/Pages/ItemFrame/UploadAvatar.cshtml.cs` (backend)

### Upload Flow
1. User selects a file via drag-drop or file picker
2. Browser `FileReader` reads file as Base64
3. POST to `OnPostStoreAnimation()` with:
   - `animationName` (e.g., "idle", "moveleft")
   - `mimeType` (image/webp, image/gif, image/png, image/jpeg)
   - `dataBase64` (base64-encoded file data)
   - `sourceInfo` (original filename)

### Server Validation
- MIME type must be one of: webp, gif, png, jpeg
- File size <= `Config.MaxUploadAvatarAnimationSize` (600 KB)
- User must own the item

### Blob Storage
**Method:** `StoreBlob()` in `AvatarItemFrameModel` (shared base class)

1. SHA256 hash of file data
2. Blob ID constructed: `{prefix}/{hash[0:2]}/{hash[2:4]}/{hash[4:6]}/{fullHash}.{ext}`
   - Upload Avatar prefix: `uploadAvatar/`
   - Custom Avatar prefix: `customAvatar/`
3. `BlobGrain.Set()` stores data with metadata:
   ```
   creationTime, mimeType, usageType (UploadAvatar or CustomAvatar), userId, sourceInfo
   ```
4. Deduplication: if blob with same hash exists, skip storage
5. Returns blob URL for the stored file

### Duration Detection
- `MediaUtils.GetMediaDurationMSec()` extracts animation duration from the uploaded file

## Phase 4: config.xml Generation

**Method:** `MakeAnimationsXml()` (line ~423)

After each animation upload/removal, the server regenerates the complete config.xml:

```xml
<config xmlns='http://schema.bluehands.de/character-config' version='1.0'>
  <param name='name' value='Avatar'/>
  <param name='width' value='100'/>
  <param name='height' value='100'/>
  <param name='defaultsequence' value='idle'/>

  <sequence group='idle' name='idle' type='status' probability='1000'
            in='standard' out='standard'>
    <animation src='{blobUrl}' duration='{ms}'/>
  </sequence>

  <sequence group='moveleft' name='moveleft' type='basic' probability='1'
            in='moveleft' out='moveleft'>
    <animation dx='-{speed}' dy='0' src='{blobUrl}' duration='{ms}'/>
  </sequence>

  <sequence group='moveright' name='moveright' type='basic' probability='1'
            in='moveright' out='moveright'>
    <animation dx='{speed}' dy='0' src='{blobUrl}' duration='{ms}'/>
  </sequence>

  <sequence group='wave' name='wave' type='emote' probability='1000'
            in='standard' out='standard'>
    <animation src='{blobUrl}' duration='{ms}'/>
  </sequence>
  <!-- ... more animations ... -->
</config>
```

### Sequence Types
| Type | Used For | Examples |
|------|----------|---------|
| `status` | Continuous states | idle |
| `basic` | Movement animations | moveleft, moveright, chat |
| `emote` | One-time actions | wave, kiss, dance, cheer, laugh |

### config.xml Storage
- The generated XML is itself stored as a blob (same `StoreBlob()` mechanism)
- `InventoryGrain.ModifyProperties()` updates the item:
  - `Pid.AvatarAnimationsUrl` = blob URL to config.xml
  - `Pid.ImageUrl` = blob URL to the icon animation

### Avatar Dimensions & Speed
- Width/height: clamped to 50-200 pixels
- Movement speed: from uploaded movement animations or default 125 px/s
- Configurable range: 10-1000 px/s (`Config.MinUploadAvatarMoveSpeed` / `MaxUploadAvatarMoveSpeed`)

### Fallback Behavior
- If moveleft/moveright/idle are not uploaded, they fall back to the idle animation
- Only icon is strictly required; all other animations are optional

## Phase 5: Avatar Activation (Client)

When the user activates the avatar (`ActivatableIsActive = true`):

1. `Participant.ts` reads `AnimationsUrl` from item properties
2. URL is wrapped through the animation proxy:
   ```
   https://webex.vulcan.weblin.com/Avatar/InlineData?url={encodedConfigUrl}
   ```
3. `avatarDisplay.updateObservableProperty('AnimationsUrl', proxiedUrl)` triggers loading

## Phase 6: Animation Proxy (WebEx Server)

**File:** `Base/n3q.WebEx/Controllers/AvatarController.cs`

### Three Proxy Endpoints

| Endpoint | Purpose |
|----------|---------|
| `/Avatar/InlineData?url=` | Fetches config.xml, inlines priority animations as base64 data URLs |
| `/Avatar/DataUrl?url=` | Converts individual images to base64 data URLs |
| `/Avatar/HttpBridge?url=` | Generic HTTP bridge for binary image data |

### InlineData Process (Primary)
1. Fetch original config.xml from blob storage URL
2. Parse XML, find all `<animation src="...">` references
3. Download animation images
4. Convert configured priority groups to data URLs:
   - Groups: `moveright`, `moveleft` (from `AvatarProxyPreloadSequenceGroups`)
   - Names: `idle` (from `AvatarProxyPreloadSequenceNames`)
5. Return modified XML with inlined base64 images
6. Cache result for 1 hour (`MemoryCacheLifetimeSec`)

### Security
- Allowed domains whitelist: webex/webit.vulcan.weblin.com, Azure blob storage, S3, IPFS, galactic-developments.de
- Max animation size: 600 KB (`MaxAnimationSizeBytes`)
- HTTPS upgrade enforced (`UpgradeAvatarXmlUrlToHttps`)

## Phase 7: Client-Side Rendering

### XML Parsing
**File:** `github-n3qExt/.../contentscript/AnimationsXml.ts`

`AnimationsXml.parseXml()` extracts:
- **params**: width, height, chatBubblesBottom, chatinBottom, defaultsequence
- **sequences**: Map of `AvatarAnimationSequence` objects:
  ```typescript
  {
    group: string,      // idle, moveleft, wave, etc.
    type: string,       // status, basic, emote
    weight: number,     // probability for random selection
    in/out: string,     // state transitions
    url: string,        // absolute URL (or data URL) to animation
    dx: number,         // pixels/sec movement speed
    duration: number,   // animation duration in ms
    loop: boolean       // whether to loop
  }
  ```

### Avatar Display
**File:** `github-n3qExt/.../contentscript/Avatar.ts`

HTML structure:
```html
<div class="entity-avatar">
  <img class="entity-avatar-image" src="..." />
</div>
```

### Animation State Machine
Four independent states with priority (highest first):

| Priority | State | Set By | Example |
|----------|-------|--------|---------|
| 1 | `currentAction` | One-time triggers | chat, wave, dance |
| 2 | `currentCondition` | XMPP presence | sleep, away |
| 3 | `currentState` | Item properties | custom states |
| 4 | `currentActivity` | Movement | moveleft, moveright |
| 5 | `defaultGroup` | Fallback | idle |

### Animation Loop (`startNextAnimation()`)
1. Select animation group based on priority
2. `getAnimationByGroup()`: collect all sequences in group, randomly select weighted by probability
3. Load image (from cache or via DataUrl proxy)
4. Set `<img src>` with random hash suffix (prevents browser GIF sync)
5. Schedule next frame via `setTimeout(duration)`
6. When the timer fires, the loop restarts at step 1 - a new random selection is made each cycle

### Weighted Random Selection (`getAnimationByGroup()`)

**File:** `github-n3qExt/.../contentscript/Avatar.ts` (line 447-480)

The algorithm for choosing among multiple animations of the same group:

1. **Collect** all sequences where `sequence.group === requestedGroup`
2. **Sum weights** (`nWeightSum`) from all matching sequences' `probability` values
3. **Random number** in range `[0, nWeightSum)` via `Math.random() * nWeightSum`
4. **Iterate** through collected animations, accumulating weights - select the animation where the cumulative sum first exceeds the random number

This is a standard weighted random selection. Each animation's chance of being selected equals `probability / totalProbabilitySum`.

#### Example: Multiple Idle Animations (business01_f)

The classic GIF avatars in `avatars/gif/002/` demonstrate this with multiple idle variants:

```xml
<sequence group='idle' name='idle'   probability='1000' ...><animation src='idle.gif'/></sequence>
<sequence group='idle' name='idle-1' probability='50'   ...><animation src='idle-1.gif'/></sequence>
<sequence group='idle' name='idle-2' probability='5'    ...><animation src='idle-2.gif'/></sequence>
<sequence group='idle' name='idle-3' probability='2'    ...><animation src='idle-3.gif'/></sequence>
<sequence group='idle' name='idle-4' probability='1'    ...><animation src='idle-4.gif'/></sequence>
```

| Sequence | Probability | Selection Chance |
|----------|------------|-----------------|
| idle | 1000 | 94.6% |
| idle-1 | 50 | 4.7% |
| idle-2 | 5 | 0.47% |
| idle-3 | 2 | 0.19% |
| idle-4 | 1 | 0.09% |
| **Total** | **1058** | **100%** |

The main idle animation plays ~95% of the time. Rare variants (e.g., scratching, yawning) appear only occasionally, making the avatar feel more alive and less robotic.

The same pattern applies to chat animations (e.g., `chat` at 1000 + `chat-1` at 100).

#### Current State: Upload Avatar

The Upload Avatar item currently only supports **one animation per type** - there is no UI to upload multiple variants for the same group. The generated config.xml therefore always has exactly one sequence per group, making the probability value irrelevant (always 100% selection).

### Animation Duration and Timing

- `duration` attribute in the XML specifies milliseconds for that animation
- `AnimationsXml.ts:103` parses it with fallback `-1` (meaning: not set)
- `Avatar.ts:413-416` applies minimum: if `duration < 100ms`, defaults to `1000ms`
- For animated formats (GIF, animated WebP), the browser handles frame-by-frame playback internally - `duration` controls when the *next animation selection* happens, not individual frames
- GIF avatars (legacy) often omit `duration`, relying on the 1s fallback
- WebP avatars (current standard) always specify explicit `duration` values

### Animation Formats

| Aspect | Legacy GIF (avatars/gif/) | Current WebP (avatars/rpm/, Upload Avatar) |
|--------|--------------------------|-------------------------------------------|
| Format | .gif | .webp (animated) |
| `duration` | Often omitted (1s fallback) | Always explicit |
| `width`/`height` | Often omitted (default 100) | Always explicit |
| Multiple variants per group | Yes (up to 5 idle, 2 chat) | No (1 per group) |
| File size | Larger | Smaller at same quality |

### Movement Rendering
**File:** `github-n3qExt/.../contentscript/Entity.ts`

- `dx` value from animation defines speed in pixels/second
- CSS transition on `left` property: `transition: left {duration}s linear`
- Duration calculated: `Math.abs(distancePixels) / speedPixelPerSec`

## Complete Data Flow Diagram

```
[Empty UploadAvatar Item in Backpack]
         |
    [1] User clicks item
         |
    [2] ItemFramePopup opens iframe
         |  (signed context token)
         v
    [3] UploadAvatar.cshtml loads in iframe
         |
    [4] User uploads animation files (Base64)
         |  POST OnPostStoreAnimation()
         v
    [5] Server validates & stores blob
         |  BlobGrain.Set() -> Azure/Orleans Storage
         |  blob ID: uploadAvatar/{hash-dirs}/{hash}.{ext}
         v
    [6] MakeAnimationsXml() generates config.xml
         |  Stored as blob too
         |  Pid.AvatarAnimationsUrl = config.xml blob URL
         v
    [7] User activates avatar
         |  Participant.ts reads AnimationsUrl
         v
    [8] Animation Proxy wraps URL
         |  /Avatar/InlineData?url={configUrl}
         v
    [9] WebEx proxy fetches config.xml
         |  Downloads & inlines priority animations as data URLs
         |  Caches result (1 hour)
         v
   [10] Client parses XML (AnimationsXml.parseXml)
         |  Extracts params + sequence map
         v
   [11] Avatar.startNextAnimation() loop
         |  Priority-based group selection
         |  Weighted random within group
         |  <img src="..."> with GIF self-animation
         v
   [12] Rendered avatar on web page
```

## config.xml Parameter Reference

The config.xml uses the namespace `http://schema.bluehands.de/character-config`.

### `<param>` Elements

| Name | Required | Description | Example |
|------|----------|-------------|---------|
| `name` | No | Avatar display name. Custom Avatar uses this as item label. | `Christine` |
| `width` | Yes | Avatar width in pixels (clamped 50-200 for Upload Avatar) | `200` |
| `height` | Yes | Avatar height in pixels (clamped 50-200 for Upload Avatar) | `200` |
| `defaultsequence` | Yes | Animation group to play by default | `idle` |
| `image` | No | Still image filename for preview (Custom Avatar only) | `still.png` |
| `chatBubblesBottom` | No | Vertical offset for chat bubbles above avatar | `100` |
| `chatinBottom` | No | Vertical offset for chat input above avatar | `35` |

### `<sequence>` Attributes

| Attribute | Required | Description |
|-----------|----------|-------------|
| `group` | Yes | Animation category (idle, moveleft, wave, etc.) |
| `name` | Yes | Unique name within config (e.g., idle, idle_1, idle_2) |
| `type` | Yes | `status` (continuous), `basic` (movement/chat), `emote` (one-time) |
| `probability` | Yes | Weight for random selection within group (higher = more frequent) |
| `in` | No | Entry transition state (`standard`, `moveleft`, `moveright`) |
| `out` | No | Exit transition state |

### `<animation>` Attributes (inside `<sequence>`)

| Attribute | Required | Description |
|-----------|----------|-------------|
| `src` | Yes | URL or filename of animation file |
| `duration` | No | Animation duration in milliseconds (client fallback: 1000ms) |
| `dx` | No | Movement speed in pixels/second (negative = left) |
| `dy` | No | Vertical movement (rarely used) |

## Shared Server Architecture

### Class Hierarchy

```
ItemFrameModel                    (base for all item frame pages)
  └── AvatarItemFrameModel        (shared avatar functionality)
        ├── UploadAvatarModel     (single animation upload)
        └── CustomAvatarModel     (ZIP upload)
```

**File:** `Items/n3q.WebIt/AvatarItemFrameModel.cs`

`AvatarItemFrameModel` provides shared methods:
- `StoreBlob(blobPrefix, usageType, mimeType, urlSuffix, blobData, sourceInfo)` - SHA256-based blob storage with deduplication
- `AddParamToXml(elem, name, value)` - XML param element helper
- `MakeErrorResult(errorId, errorMsg)` - Error response formatting
- `MakeExResult(ex)` - Exception response formatting with Orleans error detection
- Static MIME type maps: `AnimationMimeTypeFileExtensions`, `ExtensionMimeTypes`

### Blob UsageTypes

| UsageType | Item | Blob ID Prefix |
|-----------|------|----------------|
| `UploadAvatar` | Upload Avatar | `uploadAvatar/` |
| `CustomAvatar` | Custom Avatar | `customAvatar/` |

## Custom Avatar Item

### Overview

The Custom Avatar item accepts a ZIP file containing a config.xml and all referenced animation files. Unlike Upload Avatar, it supports:
- **Multiple animations per group** with different probabilities (e.g., 7 idle variants)
- **Custom config.xml** with all parameters pre-configured
- **Still image** via `<param name='image'>` for preview display
- **Batch upload** - all files in one ZIP instead of uploading one at a time

### Item Definition

**File:** `Items/n3q.Items/Templates/Standard.cs`

Same aspects as Upload Avatar: `AvatarAspect`, `ActivatableAspect`, `IframeAspect`. IframeUrl points to `ItemFrame/CustomAvatar`.

### ZIP Structure

```
avatar.zip
├── config.xml          (required - avatar configuration)
├── still.png           (optional - referenced by <param name='image'>)
├── idle.webp           (referenced by <sequence> src attributes)
├── idle_1.webp
├── idle_2.webp
├── moveleft.webp
├── moveright.webp
├── wave.webp
└── ...
```

### Example config.xml (Christine avatar)

```xml
<?xml version='1.0' encoding='UTF-8'?>
<config xmlns='http://schema.bluehands.de/character-config' version='1.0'>
    <param name='name' value='Christine'/>
    <param name='width' value='200'/>
    <param name='height' value='200'/>
    <param name='defaultsequence' value='idle'/>
    <param name='image' value='still.png'/>
    <sequence group="idle" name="idle" type="emote" probability="1000" ...><animation src="idle.webp" duration="1000"/></sequence>
    <sequence group="idle" name="idle_1" type="emote" probability="50" ...><animation src="idle_1.webp" duration="2002"/></sequence>
    <sequence group="idle" name="idle_2" type="emote" probability="200" ...><animation src="idle_2.webp" duration="1735"/></sequence>
    <!-- ... more idle variants with different probabilities ... -->
    <sequence group="moveleft" name="moveleft" type="basic" ...><animation src="moveleft.webp" dx="-109" duration="1268"/></sequence>
    <sequence group="moveright" name="moveright" type="basic" ...><animation src="moveright.webp" dx="104" duration="1268"/></sequence>
    <sequence group="wave" name="wave" type="emote" ...><animation src="wave.webp" duration="3136"/></sequence>
    <!-- ... more emotes ... -->
</config>
```

### Server Processing (Two-Phase)

**File:** `Items/n3q.WebIt/Pages/ItemFrame/CustomAvatar.cshtml.cs`

#### Phase 1: ExtractAndValidate()
Parses ZIP without side effects. Returns `ValidatedZipContent` or throws `ValidationException`:
1. Find `config.xml` in ZIP
2. Parse XML, extract all `<param>` elements
3. If `<param name='image'>` references a file, read and validate it from ZIP
4. For each `<sequence>`: find referenced animation file in ZIP, validate MIME type and size
5. Return validated data structure with all file bytes loaded

#### Phase 2: StoreAnimationsAndCreateXml()
Only runs after successful validation:
1. Store image blob (if present) → becomes `Pid.ImageUrl`
2. Store each animation blob → get blob URLs
3. Build new config.xml with blob URLs replacing relative filenames (all other attributes preserved)
4. Store config.xml blob → becomes `Pid.AvatarAnimationsUrl`
5. Set `Pid.Label` from `<param name='name'>` if present
6. Update item via `InventoryGrain.ModifyProperties()`

### Frontend UI

**File:** `Items/n3q.WebIt/Pages/ItemFrame/CustomAvatar.cshtml`

Layout:
1. **Header**: Name (editable), Image preview (from `<param name='image'>`), Active checkbox
2. **ZIP drop zone**: Drag-and-drop + click-to-upload
3. **Animation table** (after upload): Group, Name, Probability, Duration, dx - with thumbnail preview per row and hover popup showing full animation

### Configuration

| Config Key | Default | Description |
|------------|---------|-------------|
| `MaxCustomAvatarZipSize` | 10 MB | Maximum ZIP file size |
| `MaxCustomAvatarAnimationSize` | 600 KB | Maximum individual animation file size |

### Comparison: Upload Avatar vs Custom Avatar

| Feature | Upload Avatar | Custom Avatar |
|---------|--------------|---------------|
| Upload method | One animation at a time | ZIP with all files |
| config.xml | Auto-generated | User-provided in ZIP |
| Animations per type | 1 | Unlimited (with probabilities) |
| Probability control | Not applicable (always 1000) | User-defined in config.xml |
| Width/Height | Editable in UI (50-200px) | From config.xml params |
| Movement speed | Editable in UI | From `dx` in config.xml |
| Still image | Not supported | Via `<param name='image'>` |
| Avatar name | Editable label | From `<param name='name'>` |

## Key Files Reference

| Component | File |
|-----------|------|
| **Shared** | |
| Avatar base model | `Items/n3q.WebIt/AvatarItemFrameModel.cs` |
| ItemFrame base (server) | `Items/n3q.WebIt/ItemFrameModel.cs` |
| Item template definitions | `Items/n3q.Items/Templates/Standard.cs` |
| Template registry | `Items/n3q.Items/TemplateRegistry.cs` |
| Blob storage grain | `Items/n3q.Grains/BlobGrain.cs` |
| Blob grain interface | `Items/n3q.GrainInterfaces/IBlob.cs` |
| Blob proxy controller | `Items/n3q.WebIt/Controllers/BlobProxyController.cs` |
| Animation proxy | `Base/n3q.WebEx/Controllers/AvatarController.cs` |
| WebEx config | `App/n3q.AppInterfaces/WebExConfigDefinition.cs` |
| WebIt config | `App/n3q.AppInterfaces/WebItConfigDefinition.cs` |
| **Upload Avatar** | |
| Upload UI backend | `Items/n3q.WebIt/Pages/ItemFrame/UploadAvatar.cshtml.cs` |
| Upload UI frontend | `Items/n3q.WebIt/Pages/ItemFrame/UploadAvatar.cshtml` |
| **Custom Avatar** | |
| Custom UI backend | `Items/n3q.WebIt/Pages/ItemFrame/CustomAvatar.cshtml.cs` |
| Custom UI frontend | `Items/n3q.WebIt/Pages/ItemFrame/CustomAvatar.cshtml` |
| **Client (shared by both)** | |
| ItemFrame context | `github-n3qExt/.../lib/ItemFrameContextFactory.ts` |
| XML parser | `github-n3qExt/.../contentscript/AnimationsXml.ts` |
| Avatar rendering | `github-n3qExt/.../contentscript/Avatar.ts` |
| Entity movement | `github-n3qExt/.../contentscript/Entity.ts` |
| Participant integration | `github-n3qExt/.../contentscript/Participant.ts` |
| Avatar gallery | `github-n3qExt/.../lib/AvatarGallery.ts` |
| iframe API protocol | `github-n3qExt/.../lib/WeblinClientIframeApi.ts` |
| Config | `github-n3qExt/.../lib/Config.ts` (lines ~256-329) |
| User avatar grain | `Items/n3q.Grains/SetUserAvatarGrain.cs` |
