# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AEC Data Model Explorer -- an ASP.NET Core 6.0 web app that wraps GraphiQL to query the Autodesk AEC Data Model GraphQL API (`https://developer.api.autodesk.com/aec/graphql`) and integrates with the Autodesk Viewer to visualize query results on 3D models.

## Build & Run

```bash
dotnet restore              # restore NuGet packages
dotnet build                # build the project
dotnet run                  # runs on http://localhost:8080
```

No test suite exists in this repository.

## Required Configuration

Set credentials in `appsettings.Development.json` before running:
- `APS_CLIENT_ID` -- your Autodesk Platform Services app client ID
- `APS_CLIENT_SECRET` -- your APS app client secret
- `APS_CALLBACK_URL` -- must match the callback registered in your APS app (default: `http://localhost:8080/api/auth/callback`)

The app will throw at startup if any of these are missing.

## Architecture

### Backend (C# / ASP.NET Core)

- **`Startup.cs`** -- DI setup; reads APS credentials from config, registers singleton `APSService`
- **`Controllers/AuthController.cs`** -- OAuth 3-legged flow: `/api/auth/signin`, `/api/auth/callback`, `/api/auth/signout`, `/api/auth/profile`, `/api/auth/token`. Tokens stored in browser cookies; auto-refreshed when expired via `PrepareTokens`
- **`Controllers/HubsController.cs`** -- Single endpoint `GET /api/hubs/{project}/contents/{item}/versions` to list item versions
- **`Models/APSService.cs`** -- Partial class defining `Tokens` model and service constructor using `Autodesk.SDKManager`
- **`Models/APSService.Auth.cs`** -- Auth logic: 3-legged token generation/refresh, custom credential support, user profile retrieval
- **`Models/APSService.Hubs.cs`** -- Data Management API calls (get versions)

`APSService` is a **partial class** split across three files. The service supports both default and custom APS credentials (toggled via `_customCredentials` flag).

### Frontend (static files in `wwwroot/`)

The frontend is entirely static -- no build step, no bundler. CDN dependencies include React 17, GraphiQL 2.0.13, Autodesk Viewer 7.x, Bootstrap 5, SweetAlert2, and Balloon.css.

- **`index.html`** -- Main page. Renders GraphiQL with pre-configured tutorial tabs (4 tasks walking through Hubs -> Projects -> ElementGroups -> Elements). Contains inline `<script>` with `graphQLFetcher` that proxies queries to the AEC GraphQL endpoint with the user's OAuth token and selected region header
- **`main.js`** (ES module) -- App initialization: sign-in state, viewer toggle, model loading/resizing logic
- **`viewer.js`** (ES module) -- Autodesk Viewer initialization and model loading. Finds "master views" by a specific folder UUID (`08f99ae5-b8be-4f8d-881b-128675723c10`)
- **`AECDMFilterExtension.js`** -- Custom Viewer extension that parses GraphQL query responses for "External ID" values and isolates matching elements in the 3D model
- **`customcredentials.html`** -- Alternate sign-in page that lets users provide their own APS client ID/secret
- **`voyager.html`** -- GraphQL Voyager for schema visualization

### Key Data Flow

1. User signs in via OAuth -> tokens stored as cookies
2. GraphiQL sends queries to `https://developer.api.autodesk.com/aec/graphql` with bearer token + region header
3. Query responses are captured in global `queryResponse` variable
4. When Viewer is active, `AECDMFilterExtension` reads `queryResponse`, extracts External IDs, maps them to Viewer dbIds, and isolates those elements

### Global Variables (frontend)

`globalViewer` and `queryResponse` are window-level globals that bridge GraphiQL and the Viewer extension.

## NuGet Dependencies

- `Autodesk.Authentication` 1.0.0
- `Autodesk.DataManagement` 1.0.0
- `Autodesk.Forge` 1.9.9 (used for `UserProfileApi`)
