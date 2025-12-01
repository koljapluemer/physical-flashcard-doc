# Frontend Design Guidelines

- Use Tailwind and Daisy.UI components
- use lucide (installed `lucide-vue-next`) for any and all icons
- Understand [App.vue](src/app/App.vue): Note that the router view is already wrapped with a container and a flex layout. Do not wrap a page into another container or flex layout for no reason.
- In general (barring special cases where it makes no sense), every page should have a `h1` on top
- Use wrapping components and especially cards sparingly, and only when needed.
- When using a card, give it classes `card` and `shadow`. Nothing else. No variation unless called for.
- If a card must have an hover effect because it's clickable, give it `transition-hover` and `hover:shadow-md`
- A `card` always must have a `card-body` where the content lives (this is Daisy UI syntax)
- A `card-title`, if existing, must be within `card-body`
- Prefer clean `grid` and `flex` layouts over `space-*`
- Use standard buttons unless special case calls for customization. Do not vary the size randomly unless called for
- Do not use gray text. If text must be dis-emphasized, use only and consistently `text-light`. Do not give it an `sm` size.
- Do not use excessive subheadings, redundant labels or little information widgets that the user does not care about. 
- Before implementing a component, look for similar components and copy their styles and/or approach.
- When setting margins, paddings, gaps and so on, prefer the size `1`, `2`, `4`, and `6`
- For recurring complex styles, use `@apply` in `App.vue`.
- User color sparingly, and only for primary/important elements or those that must use color to communicate (e.g. a warning)
- Make sure any given layout works well on mobile and desktop!
- use primary buttons (`btn-primary`) ONLY!!!!!!!!!! for truly primary actions, of which there are very few
- use [toasts](src/stores/toast.ts) for communication with the user (error states etc.)
- Use this pattern for form inputs:

```
<fieldset class="fieldset">
  <label for="page-title" class="label">Page title</label>
  <input
    type="text"
    name="page-title"
    class="input"
    placeholder="My awesome page"
  />
</fieldset>
```

- KEEP. IT. SIMPLE.
- This is an INTERNAL EXPERT app. Avoid promoting features, marketing speak, waffling and yapping about how features work or why there are awesome. Users know what they are doing.

- Always run `npm run build` and `npm run lint` for validation.