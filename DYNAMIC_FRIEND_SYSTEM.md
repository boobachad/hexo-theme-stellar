# Dynamic Friend System / Friendship Chain System Documentation

## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture](#architecture)
3. [Backend Implementation](#backend-implementation)
4. [Frontend Implementation](#frontend-implementation)
5. [Data Format](#data-format)
6. [Configuration](#configuration)
7. [CSS Styling](#css-styling)
8. [Usage Examples](#usage-examples)

---

## System Overview

The Dynamic Friend System (also called Friendship Chain System) in Hexo Theme Stellar is a feature that displays friend links or user cards dynamically. It supports two modes:

1. **Static Mode**: Displays friends from the theme's configuration file (`_config.yml`)
2. **Dynamic Mode**: Fetches friend data from an external API endpoint

The system also supports an enhanced mode called "Friends and Posts" that displays not only friend information but also their recent blog posts by fetching RSS feeds.

---

## Architecture

The friend system consists of three main components:

### 1. Backend Tag Plugin
- **File**: `scripts/tags/lib/friends.js`
- **Purpose**: Hexo tag processor that generates HTML markup during site generation
- **Registered as**: Both `{% friends %}` and `{% users %}` tags

### 2. Frontend JavaScript Services
- **File 1**: `source/js/services/friends.js` - Basic friends display
- **File 2**: `source/js/services/friends_and_posts.js` - Friends with posts
- **Purpose**: Client-side data fetching and dynamic rendering

### 3. CSS Styling
- **File 1**: `source/css/_components/tag-plugins/friends.styl` - Basic friend cards
- **File 2**: `source/css/_components/tag-plugins/friends_posts.styl` - Friend cards with posts
- **Purpose**: Visual styling and animations

---

## Backend Implementation

### Tag Registration

The friends tag is registered in `scripts/tags/index.js`:

```javascript
hexo.extend.tag.register('users', require('./lib/friends')(hexo))
hexo.extend.tag.register('friends', require('./lib/friends')(hexo))
```

Both `{% friends %}` and `{% users %}` tags point to the same implementation.

### Tag Plugin Code (`scripts/tags/lib/friends.js`)

```javascript
/**
 * friends.js v2 | https://github.com/xaoxuu/hexo-theme-stellar/
 * Format: {% friends [group] [repo:owner/repo] [posts:true/false] [api:http] %}
 */

'use strict'

module.exports = ctx => function(args) {
  // Parse arguments: repo, api, posts are named parameters; group is unnamed
  args = ctx.args.map(args, ['repo', 'api', 'posts'], ['group'])
  
  // Get GitHub raw content host from theme config
  const host = ctx.theme.config.api_host.ghraw
  
  var api
  if (args.api) {
    // If explicit API URL is provided, use it
    api = args.api
  } else if (args.repo) {
    // If repo is provided, construct API URL from GitHub raw content
    api = `https://${host}/${args.repo}/output/v2/data.json`
  }
  
  // Determine wrapper class based on posts parameter
  var el = `<div class="tag-plugin ${args.posts ? 'users-posts-wrap' : 'users-wrap'}">`
  
  if (api) {
    // Dynamic mode: Add data-service element that will be populated by JS
    el += `<div class="data-service ds-friends${args.posts ? '_and_posts' : ''}" ${ctx.args.joinTags(args, ['size']).join(' ')} data-api="${api}"><div class="grid-box"></div></div>`
  } else if (args.group) {
    // Static mode: Render friends from config directly
    const links = ctx.theme.config.links || {}
    el += '<div class="grid-box">'
    
    // Iterate through friends in the specified group
    for (let item of (links[args.group] || [])) {
      if (item?.url && item?.title) {
        el += `<div class="grid-cell user-card">`
        el += `<a class="card-link" target="_blank" rel="external nofollow noopener noreferrer" href="${item.url}">`
        el += `<div class="lazy-box icon">`
        el += `<img class="lazy" data-src="${item.icon || item.avatar || ctx.theme.config.default.avatar}" onerror="javascript:this.removeAttribute(&quot;data-src&quot;);this.src=&quot;${ctx.theme.config.default.avatar}&quot;;"/>`
        el += `<div class="lazy-icon" style="background-image:url(&quot;${ctx.theme.config.default.loading}&quot;);"></div>`
        el += `</div>`
        el += `<div class="name">`
        el += `<span>${item.title}</span>`
        el += `</div>`
        el += `</a>`
        el += `</div>`
      }
    }
    el += '</div>'
  }
  
  el += '</div>'
  return el
}
```

### Key Components

1. **Argument Parsing**: The tag accepts multiple arguments:
   - `group`: Named group from theme config (unnamed parameter)
   - `repo`: GitHub repository in format `owner/repo`
   - `api`: Custom API endpoint URL
   - `posts`: Boolean flag to enable posts display

2. **API URL Construction**: 
   - Uses `api_host.ghraw` from config (defaults to `raw.githubusercontent.com`)
   - Constructs URL: `https://raw.githubusercontent.com/{owner}/{repo}/output/v2/data.json`

3. **Static vs Dynamic Rendering**:
   - **Static**: Reads from `theme.config.links[group]` and renders HTML immediately
   - **Dynamic**: Creates placeholder with `data-api` attribute for JS to populate

4. **CSS Classes**:
   - `users-wrap`: Basic friend cards display
   - `users-posts-wrap`: Enhanced display with posts
   - `ds-friends`: Data service for basic friends
   - `ds-friends_and_posts`: Data service for friends with posts

---

## Frontend Implementation

### Utility Functions

The system relies on utility functions defined in `layout/_partial/scripts/utils.ejs`:

#### `utils.request(el, url, callback, onFailure)`

```javascript
request: (el, url, callback, onFailure) => {
  const maxRetry = 3;
  let retryCount = 0;

  return new Promise((resolve, reject) => {
    const load = () => {
      utils.onLoading?.(el);  // Show loading indicator

      let timedOut = false;
      const timeout = setTimeout(() => {
        timedOut = true;
        console.warn('[request] 超时:', url);

        if (++retryCount >= maxRetry) {
          utils.onLoadFailure?.(el);  // Show error indicator
          onFailure?.();
          reject('请求超时');
        } else {
          setTimeout(load, 1000);  // Retry after 1 second
        }
      }, 5000);  // 5 second timeout

      fetch(url).then(resp => {
        if (timedOut) return;
        clearTimeout(timeout);

        if (!resp.ok) throw new Error('响应失败');
        return resp;
      }).then(data => {
        if (timedOut) return;
        utils.onLoadSuccess?.(el);  // Remove loading indicator
        callback(data);
        resolve(data);
      }).catch(err => {
        clearTimeout(timeout);
        console.warn('[request] 错误:', err);

        if (++retryCount >= maxRetry) {
          utils.onLoadFailure?.(el);
          onFailure?.();
          reject(err);
        } else {
          setTimeout(load, 1000);  // Retry after 1 second
        }
      });
    };

    load();
  });
}
```

**Features**:
- Automatic retry up to 3 times on failure
- 5-second timeout per request
- Loading/success/error UI indicators
- Promise-based async handling

### Basic Friends Service (`source/js/services/friends.js`)

```javascript
utils.jq(() => {
  $(function () {
    // Find all elements with ds-friends class
    const els = document.getElementsByClassName('ds-friends');
    
    for (var i = 0; i < els.length; i++) {
      const el = els[i];
      const api = el.dataset.api;
      
      if (api == null) {
        continue;
      }
      
      const default_avatar = def.avatar;
      
      // Fetch and render data
      utils.request(el, api, async resp => {
        const data = await resp.json();
        
        // Iterate through friend items
        for (let item of (data.content || data)) {
          var cell = `<div class="grid-cell user-card">`;
          cell += `<a class="card-link" target="_blank" rel="external nofollow noopener noreferrer" href="${item.html_url || item.url}">`;;
          cell += `<img src="${item.avatar_url || item.avatar || item.icon || default_avatar}" onerror="javascript:this.removeAttribute(\'data-src\');this.src=\'${default_avatar}\';"/>`;
          cell += `<div class="name image-meta">`;
          cell += `<span class="image-caption">${item.title || item.login}</span>`;
          cell += `</div>`;
          
          // Add label if present
          if (item.labels && item.labels.length > 0) {
            let label = item.labels[0];
            cell += `<div class="label" style="background:#${label.color};">${label.name}</div>`;
          }
          
          cell += `</a>`;
          cell += `</div>`;
          
          // Append to grid
          $(el).find('.grid-box').append(cell);
        }
        
        // Initialize lazy loading for images
        window.wrapLazyloadImages(el);
      });
    }
  });
});
```

**Key Features**:
1. Waits for jQuery to load via `utils.jq()`
2. Finds all `.ds-friends` elements on page
3. Fetches data from API specified in `data-api` attribute
4. Supports two data formats: `data.content` (array) or `data` (direct array)
5. Displays first label if multiple labels exist
6. Applies lazy loading to images

### Friends with Posts Service (`source/js/services/friends_and_posts.js`)

```javascript
utils.jq(() => {
  $(function () {
    // Find all elements with ds-friends_and_posts class
    const els = document.getElementsByClassName('ds-friends_and_posts');
    
    for (var i = 0; i < els.length; i++) {
      const el = els[i];
      const api = el.dataset.api;
      
      if (api == null) {
        continue;
      }
      
      const default_avatar = def.avatar;
      
      // Fetch and render data
      utils.request(el, api, async resp => {
        const data = await resp.json();
        
        for (let item of (data.content || data)) {
          var cell = `<div class="grid-cell user-post-card">`;
          
          // Avatar and user info section
          cell += `<div class="avatar-box">`;
          cell += `<a class="card-link" target="_blank" rel="external nofollow noopener noreferrer" href="${item.html_url || item.url}">`;;
          cell += `<img src="${item.avatar_url || item.avatar || item.icon || default_avatar}" onerror="javascript:this.removeAttribute(\'data-src\');this.src=\'${default_avatar}\';"/>`;
          cell += `<span class="title">${item.title || item.login}</span>`;
          cell += `</a>`;
          
          // Labels section
          cell += `<div class="labels">`;
          for (let label of item.labels) {
            // Smart color contrast calculation
            if (label.lightness > 75) {
              cell += `<div class="label" style="background:#${label.color};color:hsla(${label.hue}, ${label.saturation}%, 20%, 1);">${label.name}</div>`;
            } else if (label.saturation > 90 && label.lightness > 40) {
              cell += `<div class="label" style="background:#${label.color};color:hsla(${label.hue}, 50%, 20%, 1);">${label.name}</div>`;
            } else {
              cell += `<div class="label" style="background:#${label.color};color:white">${label.name}</div>`;
            }
          }
          cell += `</div>`;
          cell += `</div>`;
          
          // Posts preview section
          cell += `<div class="previews">`;
          
          // Description or issue number
          if (item.description) {
            cell += `<div class="desc">${item.description || item.issue_number || ''}</div>`;
          } else {
            cell += `<div class="desc">#${item.issue_number}</div>`;
          }
          
          // Posts list
          cell += `<div class="posts">`;
          if (item.posts?.length > 0) {
            for (let post of item.posts) {
              cell += `<a class="post-link" target="_blank" rel="external nofollow noopener noreferrer" href="${post.link}">`;
              cell += `<span class="title">${post.title}</span>`;
              cell += `<span class="date">${post.published}</span>`;
              cell += `</a>`;
            }
          } else {
            // Error message if no posts
            cell += `<span class="no-post">${item.feed?.length > 0 ? 'RSS 解析失败' : '未设置 RSS 链接'}</span>`;
          }
          cell += `</div>`;
          
          cell += `</div>`;
          cell += `</div>`;
          
          // Append to grid
          $(el).find('.grid-box').append(cell);
        }
        
        // Initialize lazy loading
        window.wrapLazyloadImages(el);
      });
    }
  });
});
```

**Key Features**:
1. Enhanced display with avatar, labels, description, and posts
2. Intelligent label color contrast based on HSL values
3. Displays multiple posts per friend
4. Shows helpful error messages when RSS fails or is not configured
5. Supports GitHub issue number display

---

## Data Format

### Basic Friends Data Format

The API endpoint should return JSON in one of these formats:

#### Format 1: Direct Array

```json
[
  {
    "title": "Friend Name",
    "url": "https://example.com",
    "avatar": "https://example.com/avatar.jpg",
    "icon": "https://example.com/icon.jpg",
    "avatar_url": "https://example.com/avatar.jpg",
    "html_url": "https://example.com",
    "login": "username",
    "labels": [
      {
        "name": "Active",
        "color": "00ff00"
      }
    ]
  }
]
```

#### Format 2: Content Wrapper

```json
{
  "content": [
    {
      "title": "Friend Name",
      "url": "https://example.com",
      "avatar": "https://example.com/avatar.jpg"
    }
  ]
}
```

### Field Priority and Fallbacks

The code uses the following priority for fields:

1. **URL**: `item.html_url` → `item.url`
2. **Avatar**: `item.avatar_url` → `item.avatar` → `item.icon` → default avatar
3. **Name**: `item.title` → `item.login`

### Friends with Posts Data Format

For the enhanced mode with posts:

```json
[
  {
    "title": "Friend Name",
    "url": "https://example.com",
    "html_url": "https://example.com",
    "avatar": "https://example.com/avatar.jpg",
    "description": "Friend's blog description",
    "issue_number": 123,
    "feed": "https://example.com/feed.xml",
    "labels": [
      {
        "name": "Active",
        "color": "00ff00",
        "hue": 120,
        "saturation": 100,
        "lightness": 50
      }
    ],
    "posts": [
      {
        "title": "Blog Post Title",
        "link": "https://example.com/post-1",
        "published": "2024-01-15"
      },
      {
        "title": "Another Post",
        "link": "https://example.com/post-2",
        "published": "2024-01-10"
      }
    ]
  }
]
```

### Label Color Format

Labels support enhanced color information:

- `name`: Label text
- `color`: Hex color code (without #)
- `hue`: HSL hue value (0-360)
- `saturation`: HSL saturation percentage (0-100)
- `lightness`: HSL lightness percentage (0-100)

The frontend uses these HSL values to calculate optimal text color contrast.

---

## Configuration

### Theme Configuration (`_config.yml`)

#### API Host Configuration

```yaml
api_host:
  ghapi: api.github.com
  ghraw: raw.githubusercontent.com
  gist: gist.github.com
  ghcard: github-readme-stats.vercel.app
```

- `ghraw`: Host for fetching raw files from GitHub (used for dynamic friend data)

#### Data Services Configuration

```yaml
data_services:
  friends:
    js: /js/services/friends.js
  friends_and_posts:
    js: /js/services/friends_and_posts.js
```

These services are automatically loaded when the corresponding tag is used on a page.

#### Static Friends Configuration (Optional)

```yaml
links:
  group1:
    - title: Friend 1
      url: https://example1.com
      icon: https://example1.com/avatar.jpg
      avatar: https://example1.com/avatar.jpg
    - title: Friend 2
      url: https://example2.com
      icon: https://example2.com/avatar.jpg
```

Groups can be referenced in the friends tag without an API.

#### Default Avatar Configuration

```yaml
default:
  avatar: /img/default-avatar.jpg
  loading: /img/loading.gif
```

---

## CSS Styling

### Basic Friends Styling (`source/css/_components/tag-plugins/friends.styl`)

```stylus
// Grid layout for friend cards
.users-wrap .grid-box
  display: grid
  grid-template-columns: repeat(auto-fill, minmax(80px, 1fr))

// Individual friend card
.users-wrap .user-card
  .card-link
    color: var(--text-p1)
    font-size: 10px
    font-weight: 500
    display: flex
    justify-content: flex-start
    flex-direction: column
    align-items: center
    text-align: center
    line-height: 1.2
    border-radius: $border-button
    overflow: hidden
    position: relative
    padding: 1rem 0.5rem
    
    .name
      max-width: 100%
      display: -webkit-box
      -webkit-box-orient: vertical
      overflow: hidden
      -webkit-line-clamp: 2  // Limit to 2 lines
      position: relative
    
    .lazy-box
      border-radius: 64px  // Circular avatar
      margin: 0 0 0.5rem
    
    img
      object-fit: cover
      display: block
      width: 48px
      height: 48px
  
  .label
    margin-top: 4px
    color: white
    font-size: 10px
    padding: 1px 4px
    border-radius: 16px
    font-weight: 500

// Hover effects
.users-wrap .user-card .card-link
  .lazy-box
    floatable-trans()  // Smooth transition
  
  &:hover
    background: var(--block-border)
    
    .lazy-box
      transform: scale(1.2) rotate(8deg)  // Scale and rotate avatar
      box-shadow: $boxshadow-card-float
```

**Key Features**:
1. Responsive grid with minimum 80px card width
2. Circular avatar (64px border-radius)
3. Text overflow handling with line clamping
4. Smooth hover animations (scale + rotate)
5. Uses CSS variables for theming

### Friends with Posts Styling (`source/css/_components/tag-plugins/friends_posts.styl`)

```stylus
// Grid layout - single column
.users-posts-wrap .grid-box
  display: grid
  grid-template-columns: 1fr
  grid-gap: 2px
  overflow: hidden
  border-radius: $border-card

// Individual friend + posts card
.users-posts-wrap .user-post-card
  display: grid
  grid-template-columns: 1fr 2fr  // Avatar section : Posts section
  grid-gap: 16px
  font-weight: 500
  border-radius: 4px
  background: var(--block)
  padding: 1rem
  
  // Labels styling
  .labels
    margin-top: 4px
    display: flex
    flex-wrap: wrap
    align-items: center
    justify-content: center
    pointer-events: none
    user-select: none
    
    .label
      color: white
      font-size: 12px
      padding: 2px 6px
      margin: 4px 2px 0 2px
      line-height: 1
      border-radius: 16px
  
  // Avatar section
  .avatar-box
    display: flex
    justify-content: center
    flex-direction: column
    align-items: center
    padding: 1rem
  
  .card-link
    color: var(--text-p1)
    display: flex
    justify-content: flex-start
    flex-direction: column
    align-items: center
    text-align: center
    border-radius: 4px
    position: relative
    flex-shrink: 0
    trans1 width
    
    .title
      font-size: $fs-14
      font-weight: 500
      margin-top: 1rem
      max-width: 100%
    
    img
      object-fit: cover
      display: block
      width: 64px
      height: 64px
  
  // Posts section
  .previews
    display: flex
    justify-content: space-between
    flex-direction: column
    height: 100%
    position: relative
    
    .desc
      font-size: 1.2rem
      color: var(--text-p3)
      font-weight: 400
      margin: 1rem 0
      margin-right: auto
      position: relative
      
      &:before
        position: absolute
        font-weight: 900
        font-size: 32px
        color: var(--block-border)
        top: 0
        content: '"'  // Decorative quote
        left: -16px
        trans1 all
    
    .posts
      margin: 0.5rem 0 1rem 0
      display: flex
      flex-direction: column
      align-items: flex-start
    
    .post-link
      display: flex
      flex-direction: column
      color: var(--text-p1)
      position: relative
      trans1 color
      
      &+.post-link
        margin-top: 1rem
      
      &:before
        content: ''
        position: absolute
        left: -12px
        top: 2px
        height: 'calc(%s - 2px)' % $fs-14
        width: 4px
        border-radius: 4px
        background: var(--block-border)
        trans1 all
      
      &:hover:before
        background: var(--accent)  // Highlight bar on hover
        height: 100%
        top: 0
      
      .title
        font-size: $fs-14
        font-weight: 500
        max-width: 100%
        display: -webkit-box
        -webkit-box-orient: vertical
        overflow: hidden
        -webkit-line-clamp: 2
        position: relative
      
      .date
        font-size: $fs-12
        color: var(--text-p3)
        margin-top: 0.25rem
      
      .no-post
        font-size: $fs-13
        color: var(--text-p4)

// Hover effects
.users-posts-wrap .user-post-card
  &:hover
    .desc
      &:before
        color: var(--accent)  // Highlight quote
        transform: scale(1.2) translateX(-4px)
  
  .card-link:hover
    .lazy-box
      transform: scale(1.2) rotate(8deg)
      box-shadow: $boxshadow-card-float
  
  .post-link:hover
    color: var(--accent)
```

**Key Features**:
1. Two-column layout: avatar (1/3) + posts (2/3)
2. Decorative quote mark before description
3. Vertical indicator bar for each post
4. Hover effects on description, avatar, and posts
5. Responsive text clamping
6. Smart use of CSS variables for theming

---

## Usage Examples

### Example 1: Static Friends from Config

```markdown
{% friends group1 %}
```

This reads from `theme.config.links.group1` and displays friends statically.

**Config**:
```yaml
links:
  group1:
    - title: Alice's Blog
      url: https://alice.example.com
      icon: https://alice.example.com/avatar.jpg
    - title: Bob's Site
      url: https://bob.example.com
      icon: https://bob.example.com/avatar.jpg
```

### Example 2: Dynamic Friends from GitHub Repository

```markdown
{% friends repo:owner/friend-links %}
```

This fetches data from:
`https://raw.githubusercontent.com/owner/friend-links/output/v2/data.json`

### Example 3: Dynamic Friends from Custom API

```markdown
{% friends api:https://api.example.com/friends.json %}
```

This fetches data from the specified custom API endpoint.

### Example 4: Friends with Posts

```markdown
{% friends repo:owner/friend-links posts:true %}
```

This enables the enhanced display mode showing recent blog posts from each friend.

### Example 5: Custom API with Posts

```markdown
{% friends api:https://api.example.com/friends-with-posts.json posts:true %}
```

This fetches friends and their posts from a custom API endpoint.

---

## Complete Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                          User Writes Tag                         │
│              {% friends repo:owner/repo posts:true %}            │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Hexo Build Time (Backend)                     │
│                  scripts/tags/lib/friends.js                     │
├─────────────────────────────────────────────────────────────────┤
│  1. Parse arguments (repo, api, posts, group)                    │
│  2. Construct API URL if repo provided                           │
│  3. Generate HTML with appropriate CSS classes:                  │
│     - users-posts-wrap (if posts:true)                           │
│     - ds-friends_and_posts (if posts:true)                       │
│     - data-api="..." attribute                                   │
│  4. Output HTML to page                                          │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Browser Load Time (Frontend)                 │
│            source/js/services/friends_and_posts.js               │
├─────────────────────────────────────────────────────────────────┤
│  1. Find all elements with ds-friends_and_posts class            │
│  2. For each element:                                            │
│     a. Read data-api attribute                                   │
│     b. Call utils.request(el, api, callback)                     │
│     c. Show loading indicator                                    │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API Request (utils.request)                 │
│               layout/_partial/scripts/utils.ejs                  │
├─────────────────────────────────────────────────────────────────┤
│  1. Fetch data from API URL                                      │
│  2. Retry up to 3 times on failure (1s delay)                    │
│  3. 5-second timeout per attempt                                 │
│  4. Parse JSON response                                          │
│  5. Remove loading indicator on success                          │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Render Friend Cards                       │
│            source/js/services/friends_and_posts.js               │
├─────────────────────────────────────────────────────────────────┤
│  For each friend item in response:                               │
│  1. Create user-post-card div                                    │
│  2. Render avatar section with labels                            │
│  3. Calculate label text color based on HSL                      │
│  4. Render description or issue number                           │
│  5. Render posts list with titles and dates                      │
│  6. Append card to grid-box                                      │
│  7. Initialize lazy loading for images                           │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Apply CSS Styling                        │
│    source/css/_components/tag-plugins/friends_posts.styl         │
├─────────────────────────────────────────────────────────────────┤
│  1. Grid layout: 1 column                                        │
│  2. Each card: 2 columns (avatar : posts = 1:2)                  │
│  3. Apply hover effects:                                         │
│     - Avatar: scale(1.2) rotate(8deg)                            │
│     - Quote: scale(1.2) translateX(-4px)                         │
│     - Post indicator bar: extends to full height                 │
│  4. Apply theme colors via CSS variables                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Error Handling

The system includes comprehensive error handling at multiple levels:

### 1. Backend Tag Plugin

- Falls back to static mode if no API or repo provided
- Validates that `item.url` and `item.title` exist before rendering
- Uses default avatar if no avatar provided

### 2. Frontend Request Utility

```javascript
// From utils.request
- Automatic retry: Up to 3 attempts with 1-second delay
- Timeout: 5 seconds per attempt
- Visual feedback: Loading, success, and error indicators
- Console warnings for debugging
```

### 3. Frontend Service

```javascript
// From friends_and_posts.js
- Checks if API is null before fetching
- Supports multiple data formats (data.content or data)
- Fallback chains for fields (html_url || url)
- Error messages for missing posts: "RSS 解析失败" or "未设置 RSS 链接"
- Image error handling: Removes data-src and uses default avatar
```

### 4. Label Color Calculation

```javascript
// Smart contrast calculation prevents unreadable text
if (label.lightness > 75) {
  // Light background: dark text
} else if (label.saturation > 90 && label.lightness > 40) {
  // High saturation: darker text
} else {
  // Default: white text
}
```

---

## Performance Considerations

### 1. Lazy Loading

```javascript
window.wrapLazyloadImages(el);
```

Images are lazy-loaded to improve initial page load performance.

### 2. Request Retry Logic

- Maximum 3 retries prevents infinite loops
- 1-second delay between retries prevents server hammering
- 5-second timeout per request prevents hanging

### 3. CSS Transitions

```stylus
trans1 all  // Smooth 0.3s transition
floatable-trans()  // Optimized transform transition
```

Optimized CSS transitions use GPU acceleration via `transform` property.

### 4. Grid Layout

```stylus
display: grid
grid-template-columns: repeat(auto-fill, minmax(80px, 1fr))
```

CSS Grid provides efficient responsive layout without JavaScript.

---

## Security Features

### 1. External Link Safety

```html
target="_blank" rel="external nofollow noopener noreferrer"
```

- `nofollow`: Don't pass SEO authority
- `noopener`: Prevent window.opener access
- `noreferrer`: Don't send referrer header
- `external`: Mark as external link

### 2. Image Error Handling

```javascript
onerror="javascript:this.removeAttribute('data-src');this.src='${default_avatar}';"
```

Prevents broken image display and potential XSS via image URLs.

### 3. CORS Considerations

The system uses `fetch()` which respects CORS policies. Ensure your API endpoint:
- Sets appropriate CORS headers
- Supports preflight OPTIONS requests
- Returns JSON with correct Content-Type

---

## Extensibility

The friend system can be extended in several ways:

### 1. Custom Data Service

Create a new service in `source/js/services/` following the pattern:

```javascript
utils.jq(() => {
  $(function () {
    const els = document.getElementsByClassName('ds-custom-service');
    for (var i = 0; i < els.length; i++) {
      const el = els[i];
      const api = el.dataset.api;
      if (api == null) continue;
      
      utils.request(el, api, async resp => {
        const data = await resp.json();
        // Custom rendering logic
      });
    }
  });
});
```

### 2. Custom Styling

Override styles in `source/css/_custom.styl`:

```stylus
.users-wrap .user-card
  // Your custom styles
  .card-link
    border: 2px solid var(--accent)
```

### 3. Additional Data Fields

The system is flexible and can handle additional fields in the JSON response. Simply modify the rendering logic in the service file to display them.

### 4. Alternative Data Sources

The system can fetch from any API that returns the expected JSON format:
- Static JSON files hosted on GitHub Pages
- Dynamic API endpoints
- Cloud storage (with CORS enabled)
- CDN-hosted data files

---

## Troubleshooting

### Problem: Friends not displaying

**Solution**:
1. Check browser console for JavaScript errors
2. Verify API endpoint is accessible (check CORS)
3. Verify JSON format matches expected structure
4. Check if data service JS is loaded (Network tab)

### Problem: Images not loading

**Solution**:
1. Verify image URLs are absolute and accessible
2. Check if lazy loading is properly initialized
3. Verify default avatar is configured
4. Check image error handler is working

### Problem: Posts not showing

**Solution**:
1. Verify `posts:true` parameter is set
2. Check if posts array exists in API response
3. Verify each post has `title`, `link`, and `published` fields
4. Check if RSS feed URL is correct

### Problem: Labels have wrong colors

**Solution**:
1. Verify label includes `hue`, `saturation`, and `lightness` values
2. Check if color contrast calculation is working
3. Ensure `color` field is hex without `#` prefix

### Problem: Grid layout broken

**Solution**:
1. Check if wrapper class is correct (`users-wrap` or `users-posts-wrap`)
2. Verify CSS is loaded
3. Check for conflicting custom styles
4. Inspect grid-box element in DevTools

---

## Summary

The Dynamic Friend System is a sophisticated feature that combines:

1. **Backend Tag Processing**: Generates HTML structure during Hexo build
2. **Frontend Data Fetching**: Dynamically loads friend data from APIs
3. **Smart Rendering**: Intelligently displays friends and their posts
4. **Beautiful Styling**: Responsive grid with smooth animations
5. **Error Resilience**: Comprehensive error handling and retry logic
6. **Performance**: Lazy loading and optimized CSS transitions
7. **Security**: Safe external links and CORS-aware requests

The system is designed to be flexible, allowing for both static configuration and dynamic API-driven content, making it ideal for maintaining friend links that update automatically without rebuilding the entire site.
