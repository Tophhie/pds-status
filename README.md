# PDS Status - Tophhie Social

A real-time status dashboard for monitoring the Tophhie Social Personal Data Server (PDS), built with SvelteKit and the AT Protocol.

## Overview

This application provides a comprehensive status page for the Tophhie Social PDS instance, displaying server health metrics, account information, storage usage statistics, and more. It integrates with both the AT Protocol APIs and the Tophhie Cloud API to provide detailed insights into the server's operation.

## Features

- **Real-time Service Monitoring**
  - PDS version and health status
  - Server DID (Decentralized Identifier)
  - Account count and registration status
  - Invite code requirements

- **Statistics Dashboard**
  - Total posts published this year
  - Cloudflare R2 blob storage usage
  - Heatmap data visualization

- **Account Management**
  - List all DIDs registered on the server
  - Resolve handles from DIDs using PLC directory
  - Direct links to PLC directory entries
  - Display available user domains

- **Server Information**
  - Contact information
  - Privacy policy and terms of service links
  - Available user domains

## Tech Stack

- **Framework**: [SvelteKit](https://kit.svelte.dev/) with Svelte 5
- **Language**: TypeScript
- **Styling**: TailwindCSS 4
- **APIs**:

  - [@atproto/api](https://github.com/bluesky-social/atproto) - AT Protocol client
  - Tophhie Cloud API
  - PLC Directory API

- **Build Tool**: Vite
- **Adapter**: Static adapter for deployment

## Prerequisites

- Node.js 18.x or higher
- npm, pnpm, or yarn

## Installation

1. Clone the repository:

```bash
git clone <git@github.com:Tophhie/pds-status.git>
cd pds-status-tophhie
```

1. Install dependencies:

```bash
npm install
```

1. Configure the application by editing `src/config.ts`:

```typescript
export class Config {
  static readonly PDS_URL: string = "https://your-pds-url.com";
  static readonly TOPHHIE_CLOUD_API_URL: string = "https://your-api-url.com";
}
```

## Development

Start the development server:

```bash
npm run dev
```

The application will be available at `http://localhost:5173` (or another port if 5173 is in use).

### Development with auto-reload

```bash
npm run dev -- --open
```

This will start the server and automatically open the app in your default browser.

## Building for Production

Create a production build:

```bash
npm run build
```

The static output will be generated in the `dist` directory, ready for deployment.

Preview the production build locally:

```bash
npm run preview
```

## Available Scripts

- `npm run dev` - Start development server
- `npm run build` - Build for production
- `npm run preview` - Preview production build
- `npm run check` - Run TypeScript and Svelte checks
- `npm run check:watch` - Run checks in watch mode
- `npm run format` - Format code with Prettier
- `npm run lint` - Lint and check code formatting

## Project Structure

```text
pds-status-tophhie/
├── src/
│   ├── lib/
│   │   ├── api.ts           # API functions for PDS and cloud services
│   │   └── assets/          # Static assets (favicon, etc.)
│   ├── routes/
│   │   ├── +layout.svelte   # Root layout component
│   │   └── +page.svelte     # Main status page
│   ├── app.css              # Global styles
│   ├── app.html             # HTML template
│   └── config.ts            # Application configuration
├── static/                   # Static files
├── svelte.config.js         # SvelteKit configuration
├── tailwind.config.cjs      # TailwindCSS configuration
├── tsconfig.json            # TypeScript configuration
└── vite.config.ts           # Vite configuration
```

## API Integration

### PDS Endpoints

The application queries the following PDS endpoints:

- `com.atproto.sync.listRepos` - List all repositories (accounts)
- `_health` - Health check endpoint
- `com.atproto.server.describeServer` - Server description and metadata

### External APIs

- **PLC Directory** (`https://plc.directory/`) - Resolve DIDs to handles
- **Tophhie Cloud API** - Custom endpoints for:
  - Bluesky post heatmap data
  - R2 blob storage usage statistics

## Configuration

The main configuration is located in `src/config.ts`:

```typescript
export class Config {
  static readonly PDS_URL: string = "https://tophhie.social";
  static readonly TOPHHIE_CLOUD_API_URL: string = "https://api.tophhie.cloud";
}
```

Update these values to point to your own PDS instance and API endpoints.

## Deployment

This project is configured with the static adapter for deployment to static hosting services:

### Supported Platforms

- Vercel
- Netlify
- GitHub Pages
- Cloudflare Pages
- Any static file host

The build output in the `dist` directory contains all necessary files for deployment.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

[Add your license here]

## Acknowledgments

- Built with [SvelteKit](https://kit.svelte.dev/)
- Uses the [AT Protocol](https://atproto.com/) for decentralized social networking
- Powered by [Bluesky](https://bsky.app/) infrastructure

## Support

For issues or questions about the Tophhie Social PDS, please contact the server administrator at the email provided in the status page.
