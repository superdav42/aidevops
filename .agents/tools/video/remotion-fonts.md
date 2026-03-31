---
name: fonts
mode: subagent
description: Loading Google Fonts and local fonts in Remotion
metadata:
  tags: fonts, google-fonts, typography, text
---

# Using fonts in Remotion

## Google Fonts (`@remotion/google-fonts`)

Use this when the font exists in Google Fonts. It is type-safe and blocks rendering until the font is ready.

```bash
npx remotion add @remotion/google-fonts
bunx remotion add @remotion/google-fonts
yarn remotion add @remotion/google-fonts
pnpm exec remotion add @remotion/google-fonts
```

```tsx
import { loadFont } from "@remotion/google-fonts/Roboto";

const { fontFamily, waitUntilDone } = loadFont("normal", {
  weights: ["400", "700"],
  subsets: ["latin"],
});

await waitUntilDone(); // Needed before measuring text or DOM layout

export const Title: React.FC<{ text: string }> = ({ text }) => (
  <h1 style={{ fontFamily, fontSize: 80, fontWeight: "bold" }}>{text}</h1>
);
```

Specify only the weights and subsets you need to keep bundle size down.

## Local fonts (`@remotion/fonts`)

Use this for custom font files. Put the files in `public/` and load them at module scope, not during render.

```bash
npx remotion add @remotion/fonts
bunx remotion add @remotion/fonts
yarn remotion add @remotion/fonts
pnpm exec remotion add @remotion/fonts
```

```tsx
import { loadFont } from "@remotion/fonts";
import { staticFile } from "remotion";

await loadFont({
  family: "MyFont",
  url: staticFile("MyFont-Regular.woff2"),
});

await Promise.all([
  loadFont({ family: "Inter", url: staticFile("Inter-Regular.woff2"), weight: "400" }),
  loadFont({ family: "Inter", url: staticFile("Inter-Bold.woff2"), weight: "700" }),
]);

export const MyComposition = () => <div style={{ fontFamily: "MyFont" }}>Hello World</div>;
```

`loadFont()` accepts:

```tsx
loadFont({
  family: "MyFont",           // Required CSS font-family name
  url: staticFile("f.woff2"), // Required font file URL
  format: "woff2",            // Optional, inferred from extension by default
  weight: "400",              // Optional font weight
  style: "normal",            // Optional normal | italic
  display: "block",           // Optional font-display value
});
```
