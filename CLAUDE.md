# CLAUDE.md - FlixD Project Guide for AI Assistants

This document provides comprehensive information about the FlixD codebase structure, development workflows, and conventions for AI assistants working with this project.

---

## Project Overview

**FlixD** is a single-file, browser-based video streaming application that enables users to watch movies and TV series stored in their Dropbox app folder. The application features a modern, dark-themed interface optimized for both desktop browsers and TV remote control navigation.

### Key Capabilities
- Browse and play movies and TV series from Dropbox storage
- Track watch history with automatic progress synchronization
- Resume playback from where you left off
- Auto-play next episode in series
- TV remote and keyboard-friendly navigation
- Touch and mouse gesture support
- Lazy-loaded thumbnails and descriptions
- Context menu actions (mark as watched, remove from history)

### Technology Stack
- **Frontend**: Pure HTML5, CSS3, and vanilla JavaScript (no frameworks)
- **Storage**: Dropbox API v2 for file management and sync
- **Local Storage**: Browser localStorage for offline caching
- **Video**: HTML5 native video element with subtitle support
- **Authentication**: OAuth2 with refresh token flow

---

## Repository Structure

```
FlixD/
├── README.md           # Minimal project readme
├── flixd.html         # Main application file (single-file app)
├── CLAUDE.md          # This file - AI assistant guide
└── .git/              # Git repository data
```

### Single-File Architecture

The entire application is contained within `flixd.html` (2743 lines):

- **Lines 1-848**: CSS styling (dark theme, glassmorphism effects, responsive layouts)
- **Lines 850-863**: Configuration constants (Dropbox API credentials)
- **Lines 866-926**: HTML structure (minimal DOM templates)
- **Lines 928-2739**: JavaScript application logic

---

## File Organization in Dropbox

FlixD expects content in the Dropbox app folder to follow this structure:

```
/Movies/
  Movie Name/
    movie.mp4
    thumbnail.jpg (optional)
    description.txt (optional)

/Series/
  Show Name/
    thumbnail.jpg (optional)
    description.txt (optional)
    Season 1/
      S01E01.mp4
      S01E02.mp4
    Season 2/
      S02E01.mp4
```

### File Naming Conventions
- Movies: Any `.mp4`, `.mkv`, `.avi`, `.mov`, `.webm`, `.m4v` file
- Series: Episode naming should include season/episode identifiers (e.g., S01E01)
- Thumbnails: `thumbnail.jpg` in the same folder as the content
- Descriptions: `description.txt` in the same folder as the content

---

## Core Data Structures

### 1. Configuration Object
```javascript
const CONFIG = {
  appKey: 'ral76dcspmytq72',           // Dropbox App Key
  appSecret: 'fy0wzhlm4r8zifg',        // Dropbox App Secret
  refreshToken: 'REFRESH_TOKEN_HERE',  // OAuth refresh token
  folderPath: ''                       // Root path in Dropbox app folder
};
```

**Location**: flixd.html:857-862

**Security Note**: These credentials should be regenerated and not committed to public repositories.

### 2. Videos Array
Stores all video files discovered in Dropbox.

```javascript
let videos = [
  {
    id: 'file_id_hash',                    // Unique Dropbox file ID
    name: 'filename.mp4',                  // File name
    path: '/movies/movie name/filename.mp4', // Lowercase path
    pathDisplay: '/Movies/Movie Name/filename.mp4', // Display path
    size: 1234567890                       // File size in bytes
  }
];
```

### 3. Movies Array
Processed movie objects with metadata.

```javascript
let movies = [
  {
    id: 'file_id',
    name: 'movie.mp4',
    path: '/movies/...',
    folder: 'Movie Name',
    displayName: 'Movie Name',
    thumbnailPath: '/movies/movie name/thumbnail.jpg',
    descriptionPath: '/movies/movie name/description.txt'
  }
];
```

### 4. Series Object
Hierarchical structure for TV shows with seasons and episodes.

```javascript
let series = {
  'Show Name': {
    seasons: {
      'Season 1': [
        { id: '...', name: 'S01E01.mp4', path: '...', episodeNumber: 1 },
        { id: '...', name: 'S01E02.mp4', path: '...', episodeNumber: 2 }
      ],
      'Season 2': [ /* episodes */ ]
    },
    thumbnailPath: '/series/show name/thumbnail.jpg',
    descriptionPath: '/series/show name/description.txt'
  }
};
```

### 5. Watch History Object
Tracks playback progress for all videos.

```javascript
let watchHistory = {
  'file_id_1': {
    position: 1234.5,        // Current playback position (seconds)
    duration: 5678.9,        // Total video duration (seconds)
    lastWatched: 1674591234  // Unix timestamp
  }
};
```

**Storage**:
- Primary: `/flixd-data.json` in Dropbox
- Fallback: `localStorage['dropbox_video_progress']`

---

## Key Functions Reference

### Authentication & Token Management

#### `getAccessToken()`
**Purpose**: Retrieves or refreshes Dropbox access token
**Location**: flixd.html:~970-1010
**Behavior**: Checks for cached token, refreshes if expired (5-min buffer)
**Returns**: Valid access token string

### Data Fetching

#### `listVideos()`
**Purpose**: Fetches all video files from Dropbox, organizes into movies/series
**Location**: flixd.html:~1020-1150
**Side Effects**: Populates `videos`, `movies`, `series` arrays
**Error Handling**: Shows toast notification on failure

#### `getThumbnailUrl(path)`
**Purpose**: Gets temporary download link for thumbnail images
**Parameters**: `path` - Dropbox path to thumbnail
**Returns**: Promise<string> - Temporary URL (valid for 4 hours)

#### `getDescription(path)`
**Purpose**: Fetches and returns description text from description.txt
**Parameters**: `path` - Dropbox path to description file
**Returns**: Promise<string> - Description text or empty string

#### `getStreamUrl(path)`
**Purpose**: Generates Dropbox streaming URL for video playback
**Parameters**: `path` - Dropbox path to video file
**Returns**: string - Direct streaming URL

### Watch History Management

#### `loadWatchHistoryFromDropbox()`
**Purpose**: Syncs watch history from Dropbox `/flixd-data.json`
**Location**: flixd.html:~1160-1200
**Fallback**: Uses localStorage if Dropbox fetch fails

#### `saveWatchHistoryToDropbox()`
**Purpose**: Persists watch history back to Dropbox
**Location**: flixd.html:~1210-1250
**Behavior**: Writes JSON to Dropbox, also saves to localStorage

#### `updateProgress(fileId, position, duration)`
**Purpose**: Records current playback position for a video
**Parameters**:
  - `fileId` - Unique video identifier
  - `position` - Current time in seconds
  - `duration` - Total duration in seconds
**Side Effects**: Updates `watchHistory`, triggers save

#### `getProgress(fileId)`
**Purpose**: Retrieves watch history for a specific video
**Returns**: `{position, duration, lastWatched}` or null

#### `removeFromHistory(fileId)`
**Purpose**: Deletes a video from watch history
**Use Case**: "Remove from Up Next" action

#### `markAsComplete(fileId)`
**Purpose**: Marks video as fully watched (100% progress)
**Use Case**: "Mark as Watched" context menu action

### Rendering Functions

#### `render()`
**Purpose**: Main render dispatcher based on current view
**Location**: flixd.html:~1350
**Logic**: Calls `renderHome()` or `renderSeriesDetail()` depending on `currentView`

#### `renderHome(app)`
**Purpose**: Renders home view with Up Next, Movies, and Series sections
**Location**: flixd.html:~1360-1450
**Side Effects**: Updates DOM, calls card renderers

#### `renderSeriesDetail(app)`
**Purpose**: Renders series detail view with seasons and episodes
**Location**: flixd.html:~1550-1650
**Features**: Season tabs, episode list, back navigation

#### `renderMovieCard(movie)`
**Purpose**: Generates HTML for a movie card
**Returns**: HTML string with thumbnail, title, progress bar
**Features**: Lazy-loading, progress visualization

#### `renderSeriesCard(seriesName)`
**Purpose**: Generates HTML for a series card
**Returns**: HTML string with season/episode counts

#### `renderEpisodeItem(episode, episodeNumber)`
**Purpose**: Generates HTML for an episode in series detail view
**Returns**: HTML string with episode info, progress, watch status

### Playback Control

#### `playVideo(fileId)`
**Purpose**: Loads and plays a video
**Location**: flixd.html:~1900-1990
**Behavior**:
  - Finds video by ID
  - Gets streaming URL
  - Restores saved position
  - Auto-plays video
  - Saves progress every 3 seconds
  - Sets up ended event for auto-play next

#### `closePlayer()`
**Purpose**: Exits player and returns to previous view
**Location**: flixd.html:~2000-2020
**Behavior**: Saves final position, restores focus, renders previous view

#### `getNextEpisode(video)`
**Purpose**: Determines next episode in sequence
**Location**: flixd.html:~2100-2150
**Logic**: Finds next episode in season, or first episode of next season
**Returns**: Episode object or null

### Event Handlers

#### `attachCardEventHandlers()`
**Purpose**: Binds click/keyboard/context menu events to video cards
**Location**: flixd.html:~1700-1800
**Features**: Hold-to-remove gesture, context menu, focus management

#### `attachEpisodeEventHandlers()`
**Purpose**: Binds events to episode items in series detail view
**Location**: flixd.html:~1810-1850

### Utility Functions

#### `stripExtension(filename)`
**Purpose**: Removes file extension from filename
**Example**: `"movie.mp4"` → `"movie"`

#### `formatTime(seconds)`
**Purpose**: Converts seconds to MM:SS format
**Example**: `125` → `"02:05"`

#### `escapeHtml(text)`
**Purpose**: Sanitizes text to prevent XSS
**Important**: Always use this when displaying user-generated content

#### `showToast(message, isError)`
**Purpose**: Shows temporary notification (3 seconds)
**Parameters**:
  - `message` - Text to display
  - `isError` - Boolean, shows red background if true

---

## Development Workflows

### Making Code Changes

1. **Read Before Modifying**: Always read the relevant section of `flixd.html` before making changes
2. **Test in Browser**: Changes require browser refresh to test
3. **Check Console**: Use browser DevTools console for debugging
4. **Validate Dropbox API**: Ensure API calls work with test credentials

### Common Modification Scenarios

#### 1. Updating Styles
**Location**: Lines 1-848 in flixd.html
**Pattern**: CSS is organized by component (clock, container, video-card, player, etc.)
**Testing**: Refresh browser and check visual changes

#### 2. Modifying Dropbox Configuration
**Location**: Lines 857-862 in flixd.html
**Security**: Never commit real credentials to public repos
**Testing**: Verify token refresh and file listing work

#### 3. Adding New Features
**Approach**:
1. Add any required state variables at top of script section
2. Implement core logic functions
3. Add UI rendering code
4. Attach event handlers
5. Update `init()` if needed

#### 4. Fixing Bugs
**Steps**:
1. Identify the affected function(s) using line numbers
2. Add console.log statements for debugging
3. Test in browser with DevTools open
4. Verify fix doesn't break related features

### Testing Checklist

Before committing changes, verify:
- [ ] Application loads without console errors
- [ ] Video playback works
- [ ] Watch history saves and restores
- [ ] Navigation works (keyboard, mouse, touch)
- [ ] Context menu functions correctly
- [ ] Series detail view displays properly
- [ ] Auto-play next episode works
- [ ] Hold-to-remove gesture functions
- [ ] Thumbnails lazy-load properly

---

## Key Conventions for AI Assistants

### Code Style

1. **Indentation**: 2 spaces (consistent throughout file)
2. **Quotes**: Single quotes for JavaScript strings
3. **Semicolons**: Used consistently
4. **Variable Naming**: camelCase for variables and functions
5. **Constants**: UPPER_SNAKE_CASE (e.g., `CONFIG`, `STORAGE_KEY`)

### Function Organization Pattern

Functions are grouped by purpose:
1. Configuration and constants
2. Authentication functions
3. Data fetching functions
4. Watch history functions
5. Rendering functions
6. Event handler functions
7. Playback control functions
8. Utility/helper functions
9. Initialization function

### DOM Manipulation

- **No jQuery**: Uses vanilla JavaScript `querySelector`, `innerHTML`
- **Template Literals**: HTML is built with template strings
- **Event Delegation**: Events attached after DOM updates
- **Focus Management**: Tracks and restores focus state

### State Management

- **Global Variables**: State stored in module-level variables
- **No Framework**: Pure JavaScript state management
- **Reactivity**: Manual DOM updates via `render()` calls
- **Persistence**: State synced to Dropbox + localStorage

### Error Handling

- **Try-Catch**: Wrap all async operations
- **User Feedback**: Show toast notifications for errors
- **Graceful Degradation**: Fall back to localStorage if Dropbox fails
- **Console Logging**: Log errors for debugging

### Security Considerations

1. **XSS Prevention**: Always use `escapeHtml()` for user content
2. **API Credentials**: Never commit real tokens to repository
3. **Token Expiry**: Auto-refresh tokens before expiry
4. **CORS**: Dropbox API handles CORS properly
5. **Content Security**: Only play video from Dropbox URLs

---

## Common Tasks Guide

### Task: Add New Keyboard Shortcut

1. Locate keyboard event handler (search for `keydown` event listener)
2. Add new case in the switch statement
3. Implement shortcut logic
4. Update any UI documentation if needed

**Example Location**: flixd.html:~2300-2400

### Task: Modify Video Card Layout

1. Find `renderMovieCard()` or `renderSeriesCard()` function
2. Update HTML template string
3. Modify corresponding CSS in the `<style>` section
4. Test visual changes in browser

### Task: Change Watch History Sync Behavior

1. Locate `saveWatchHistoryToDropbox()` function
2. Modify save logic (currently saves every 3 seconds during playback)
3. Update `updateProgress()` calls if needed
4. Test that history persists correctly

### Task: Add New Video Format Support

1. Find supported extensions array (search for `.mp4`, `.mkv`)
2. Add new extension to the array
3. Test with sample file in that format
4. Verify Dropbox and browser both support the format

### Task: Customize UI Theme

1. Locate CSS variables or color values in `<style>` section
2. Update color codes (currently uses dark theme with purple/pink accents)
3. Check all UI states (hover, focus, active, disabled)
4. Test in both desktop and TV remote mode

### Task: Modify Auto-Play Behavior

1. Find `playVideo()` function and video `ended` event handler
2. Locate countdown timer logic (currently 5 seconds)
3. Update timer duration or add user preference
4. Test with consecutive episodes

### Task: Add New Context Menu Option

1. Find `openContextMenu()` function
2. Add new menu item HTML
3. Implement action handler in event listener
4. Add corresponding function logic
5. Test right-click and keyboard ('c' key) activation

---

## API Integration Details

### Dropbox API Endpoints Used

| Endpoint | Purpose | Frequency |
|----------|---------|-----------|
| `/2/oauth2/token` | Refresh access token | Every ~55 minutes |
| `/2/files/list_folder` | List videos in folder | On app load, manual refresh |
| `/2/files/list_folder/continue` | Pagination for large folders | As needed during listing |
| `/2/files/get_temporary_link` | Get thumbnail/description URLs | On demand (4-hour cache) |
| `/2/files/download` | Download watch history JSON | On app load |
| `/2/files/upload` | Save watch history JSON | Every 3 seconds during playback |

### OAuth2 Flow

1. User generates refresh token via Dropbox OAuth flow (external)
2. App stores refresh token in `CONFIG.refreshToken`
3. On startup, app exchanges refresh token for access token
4. Access token cached in localStorage with expiry timestamp
5. Token refreshed automatically when within 5 minutes of expiry

### Rate Limiting Considerations

- Dropbox API has rate limits (not specified in code)
- Thumbnail URLs cached (4-hour expiry from Dropbox)
- Watch history saves throttled to every 3 seconds
- Manual refresh should not be spammed

---

## Debugging Guide

### Common Issues and Solutions

#### Videos Not Loading
1. Check browser console for API errors
2. Verify Dropbox credentials in CONFIG
3. Ensure video files exist in correct folder structure
4. Check network tab for failed requests

#### Watch History Not Saving
1. Verify localStorage is enabled
2. Check Dropbox write permissions
3. Look for `saveWatchHistoryToDropbox()` errors in console
4. Test localStorage fallback

#### Thumbnails Not Displaying
1. Confirm `thumbnail.jpg` exists in correct location
2. Check `getThumbnailUrl()` for API errors
3. Verify image file is valid JPEG
4. Test with browser network tab

#### Video Playback Issues
1. Verify video format is supported (MP4, MKV, etc.)
2. Check browser console for codec errors
3. Test video file plays in standalone player
4. Ensure Dropbox URL is valid

#### Focus/Navigation Problems
1. Check `lastFocusedCardId` is being tracked
2. Verify `attachCardEventHandlers()` is called after render
3. Test with keyboard navigation step by step
4. Check for JavaScript errors blocking event handlers

### DevTools Tips

- **Console**: Check for API errors, state variables
- **Network**: Monitor Dropbox API calls and responses
- **Application**: Inspect localStorage values
- **Elements**: Debug CSS and DOM structure
- **Sources**: Set breakpoints in JavaScript code

---

## Git Workflow

### Branch Naming Convention

Follow the pattern: `claude/claude-md-<session-id>-<random>`

**Example**: `claude/claude-md-mkp8qr8u070vlm9k-bR0O1`

### Commit Message Guidelines

- Use clear, descriptive commit messages
- Start with imperative verb (Add, Update, Fix, Remove, etc.)
- Reference issue numbers if applicable
- Keep messages concise but informative

**Examples**:
```
Add episode auto-play countdown feature
Fix watch history sync on page refresh
Update thumbnail lazy-loading behavior
Remove deprecated sorting function
```

### Push Protocol

1. Always push to branch specified in task instructions
2. Use: `git push -u origin <branch-name>`
3. Retry on network errors with exponential backoff (2s, 4s, 8s, 16s)
4. Never force-push to main/master branches

---

## Project-Specific Considerations

### Single-File Constraints

- **No Module System**: All code in one file, no imports/exports
- **No Build Process**: Direct browser execution
- **Large File**: 2700+ lines, use line numbers for navigation
- **No Package Manager**: No npm, no dependencies
- **No TypeScript**: Pure JavaScript, no type checking

### Performance Characteristics

- **Initial Load**: Fetches all videos on startup (can be slow for large libraries)
- **Thumbnail Loading**: Lazy-loaded to reduce initial bandwidth
- **Watch History**: Synced frequently during playback (3-second intervals)
- **Token Refresh**: Minimal overhead, checked every API call

### Browser Compatibility

- **Target**: Modern browsers (Chrome, Firefox, Safari, Edge)
- **Features Used**: ES6+, async/await, template literals, fetch API
- **Fallbacks**: localStorage if Dropbox sync fails
- **TV Browsers**: Optimized for TV remote navigation

### Accessibility Features

- **Keyboard Navigation**: Full support with arrow keys, enter, escape
- **TV Remote**: D-pad navigation optimized
- **Focus Indicators**: Visual focus states for all interactive elements
- **Large Touch Targets**: Optimized for TV and touch devices
- **Screen Reader**: Minimal support (could be improved)

---

## Future Enhancement Ideas

When asked to add features, consider these common requests:

1. **Multi-user Support**: Separate watch history per user
2. **Search Functionality**: Filter movies/series by name
3. **Sorting Options**: By name, date added, recently watched
4. **Playlist Creation**: Custom playlists for queuing content
5. **Cast Support**: Chromecast/AirPlay integration
6. **Subtitle Management**: Upload and manage subtitle files
7. **Settings Panel**: Configurable options (theme, auto-play, etc.)
8. **Offline Mode**: Service worker for offline viewing
9. **Multiple Storage**: Support for Google Drive, OneDrive
10. **Mobile App**: PWA manifest for mobile installation

---

## Resources and References

### External Documentation

- **Dropbox API v2**: https://www.dropbox.com/developers/documentation/http/documentation
- **HTML5 Video**: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/video
- **Web Storage API**: https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API
- **OAuth 2.0**: https://oauth.net/2/

### Code Sections Quick Reference

| Feature | Line Range |
|---------|------------|
| CSS Styles | 1-848 |
| Dropbox Config | 857-862 |
| HTML Structure | 866-926 |
| Authentication | ~970-1010 |
| Data Fetching | ~1020-1200 |
| Watch History | ~1210-1330 |
| Rendering | ~1350-1680 |
| Event Handlers | ~1700-1900 |
| Playback Control | ~1900-2200 |
| Utilities | ~2210-2300 |
| Keyboard Handling | ~2300-2550 |
| Initialization | ~2700-2739 |

---

## Summary for AI Assistants

When working with FlixD:

1. **Understand the single-file architecture**: All code is in `flixd.html`
2. **Use line numbers**: Navigate using line references (e.g., flixd.html:1234)
3. **Read before modifying**: Always read relevant sections before changes
4. **Test in browser**: Changes require browser refresh to test
5. **Preserve patterns**: Follow existing code style and organization
6. **Handle errors gracefully**: Use try-catch and show user feedback
7. **Consider accessibility**: Maintain keyboard and TV remote support
8. **Sync watch history**: Ensure changes don't break Dropbox sync
9. **Document changes**: Update this file if adding new major features
10. **Security first**: Never commit real API credentials

**Last Updated**: 2026-01-22
**Codebase Version**: Initial analysis
**Total Lines**: 2743 (flixd.html)
