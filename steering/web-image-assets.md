---
inclusion: manual
---

# Web Image Asset Preparation

Lessons learned from sourcing, processing, and displaying third-party badge/logo images on a dark-themed landing page.

## Sourcing images from third-party sites

- **JS-rendered pages can't be scraped directly** — tools like `web_fetch` return empty or minimal content for pages that load images via JavaScript. Go to the individual product/detail page rather than a listing page, which is more likely to have the image URL in the rendered HTML.
- **`curl` works where Python `urllib` fails on macOS** — Python's SSL stack on macOS often fails with `CERTIFICATE_VERIFY_FAILED` for external HTTPS URLs. Use `curl -sSL` to download files, then process them with Python locally.
- **Check the image URL pattern first** — once you find one badge URL, the rest usually follow the same path structure. Verify one URL returns HTTP 200 before scripting the rest.

## Background removal

- **Flood-fill from corners, not a global threshold** — replacing all near-white pixels globally will destroy white elements inside the artwork (text, highlights, borders). Instead, BFS flood-fill from the four corners of the image: only pixels reachable from the edge that meet the threshold get made transparent. This preserves interior white.
- **Threshold around 235–240** works well for AWS-style badges that have a clean white background. Lower it if the background is off-white or cream.
- **Always convert to RGBA before processing** — source images may be RGB or P (palette) mode. `img.convert("RGBA")` normalises this before you touch pixel data.

```python
from collections import deque
from PIL import Image

def remove_white_background(img: Image.Image, threshold: int = 235) -> Image.Image:
    img = img.convert("RGBA")
    data = img.load()
    w, h = img.size
    visited = [[False] * h for _ in range(w)]
    queue = deque()
    for cx, cy in [(0,0),(w-1,0),(0,h-1),(w-1,h-1)]:
        r, g, b, a = data[cx, cy]
        if r >= threshold and g >= threshold and b >= threshold:
            visited[cx][cy] = True
            queue.append((cx, cy))
    while queue:
        x, y = queue.popleft()
        r, g, b, a = data[x, y]
        if r >= threshold and g >= threshold and b >= threshold:
            data[x, y] = (r, g, b, 0)
            for dx, dy in ((-1,0),(1,0),(0,-1),(0,1)):
                nx, ny = x+dx, y+dy
                if 0 <= nx < w and 0 <= ny < h and not visited[nx][ny]:
                    visited[nx][ny] = True
                    queue.append((nx, ny))
    return img
```

## Auto-cropping transparent padding

- **Source images often have large transparent margins** after background removal — the actual artwork may only occupy 50–60% of the canvas. This makes images appear tiny at a given CSS size.
- **Use `Image.getbbox()`** to find the bounding box of non-transparent pixels, crop to it, then add a small uniform padding (8–16 px) so the artwork isn't flush with the edge.

```python
def crop_to_content(img: Image.Image, padding: int = 8) -> Image.Image:
    bbox = img.getbbox()  # (left, upper, right, lower) of non-zero pixels
    if bbox is None:
        return img  # fully transparent
    cropped = img.crop(bbox)
    cw, ch = cropped.size
    padded = Image.new("RGBA", (cw + padding*2, ch + padding*2), (0, 0, 0, 0))
    padded.paste(cropped, (padding, padding))
    return padded
```

- **Always check the before/after dimensions** — a 500×500 image cropping down to 270×308 means the badge was using only ~54% of the canvas. At the same CSS display size, the artwork will appear ~85% larger after cropping.

## Serving processed images

- **Store processed images in `public/badges/`** (or equivalent static assets folder) so the build tool serves them without hashing. This makes the HTML paths stable (`/badges/clf.png`) and avoids re-downloading on every build.
- **Keep the processing scripts** in `scripts/` so they can be re-run if source images are updated. Don't commit raw downloaded files — only commit the processed transparent PNGs.
- **Add `public/badges/*-raw.png` to `.gitignore`** if you use a two-step download-then-process workflow, so intermediate files aren't accidentally committed.

## CSS for transparent badge images on dark backgrounds

- Use `filter: drop-shadow(...)` rather than `box-shadow` on the container — `drop-shadow` follows the alpha channel of the PNG so the shadow hugs the badge shape rather than the rectangular bounding box.
- On hover, increase the drop-shadow spread and add a colour tint to make the badge glow:

```css
.badge-img {
  filter: drop-shadow(0 4px 12px rgba(0, 0, 0, 0.5));
  transition: filter 250ms ease, transform 250ms ease;
}
.badge-img:hover {
  filter: drop-shadow(0 6px 18px rgba(0, 191, 255, 0.3));
  transform: scale(1.05);
}
```

- Set `object-fit: contain` on the `<img>` so the badge scales proportionally within its CSS box regardless of the cropped aspect ratio.
