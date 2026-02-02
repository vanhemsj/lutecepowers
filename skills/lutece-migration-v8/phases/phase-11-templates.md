# Phase 9: Template & Plugin Descriptor Migration

## Reference Sources — MANDATORY

Before modifying any template, ALWAYS consult:
- **Admin macros**: `~/.lutece-references/lutece-core/webapp/WEB-INF/templates/admin/themes/tabler/` — read the macro definitions to understand available parameters
- **Admin template examples**: search `~/.lutece-references/` for real usage of `@pageContainer`, `@tform`, `@table`, etc.

---

## 9.1 Admin (Back-Office) Templates — Rewrite to v8 Macros

Every admin template (`templates/admin/**/*.html`) MUST be rewritten to use the v8 Freemarker macro system. Do NOT keep raw Bootstrap 3/4 HTML — replace with macros.

### Layout structure (MANDATORY)

Every admin page MUST use: `@pageContainer` > `@pageColumn` > `@pageHeader`.

Do NOT use `@row` / `@columns` outside of `@pageColumn`. `@row` and `@columns` can only be used **inside** `@pageColumn`.

### List page pattern

```html
<!-- BEFORE (v7) — raw Bootstrap panels/tables -->
<div class="panel panel-default">
    <div class="panel-heading"><h3 class="panel-title">Title</h3></div>
    <div class="panel-body">
        <table class="table table-striped">...</table>
    </div>
</div>

<!-- AFTER (v8) — Freemarker macros -->
<@pageContainer>
    <@pageColumn>
        <@pageHeader title='#i18n{myplugin.manage_items.pageTitle}'>
            <@aButton href='jsp/admin/plugins/myplugin/ManageItems.jsp?view=createItem' buttonIcon='plus' title='#i18n{myplugin.manage_items.buttonAdd}' color='primary' />
        </@pageHeader>
        <#if item_list?size gt 0>
            <@table>
                <tr>
                    <th>#i18n{myplugin.model.entity.item.attribute.name}</th>
                    <th>#i18n{portal.util.labelActions}</th>
                </tr>
                <#list item_list as item>
                <tr>
                    <td>${item.name!}</td>
                    <td>
                        <@aButton href='jsp/admin/plugins/myplugin/ManageItems.jsp?view=modifyItem&id=${item.id}' buttonIcon='edit' title='#i18n{portal.util.labelModify}' />
                        <@aButton href='jsp/admin/plugins/myplugin/ManageItems.jsp?action=confirmRemoveItem&id=${item.id}' buttonIcon='trash' color='danger' title='#i18n{portal.util.labelDelete}' />
                    </td>
                </tr>
                </#list>
            </@table>
        <#else>
            <@alert color='info'>#i18n{myplugin.manage_items.noData}</@alert>
        </#if>
    </@pageColumn>
</@pageContainer>
```

### Form page pattern

```html
<!-- BEFORE (v7) — raw HTML forms -->
<form method="post" action="...">
    <div class="form-group">
        <label>Title</label>
        <input type="text" class="form-control" name="title" value="${title}" />
    </div>
    <button type="submit" class="btn btn-primary">Save</button>
</form>

<!-- AFTER (v8) — Freemarker macros -->
<@pageContainer>
    <@pageColumn>
        <@pageHeader title='#i18n{myplugin.create_item.pageTitle}'>
            <@aButton href='jsp/admin/plugins/myplugin/ManageItems.jsp?view=manageItems' buttonIcon='arrow-left' title='#i18n{portal.util.labelBack}' />
        </@pageHeader>
        <@tform method='post' name='create_item' action='jsp/admin/plugins/myplugin/ManageItems.jsp' boxed=true>
            <@input type='hidden' name='action' value='createItem' />
            <@formGroup labelFor='title' labelKey='#i18n{myplugin.model.entity.item.attribute.title}' mandatory=true rows=2>
                <@input type='text' name='title' id='title' value='${item.title!}' />
            </@formGroup>
            <@formGroup labelFor='description' labelKey='#i18n{myplugin.model.entity.item.attribute.description}' rows=2>
                <@input type='textarea' name='description' id='description'>${item.description!}</@input>
            </@formGroup>
            <@formGroup labelFor='active' labelKey='#i18n{myplugin.model.entity.item.attribute.active}' rows=2>
                <@checkBox orientation='switch' labelKey='#i18n{myplugin.model.entity.item.attribute.active}' name='active' id='active' value='true' checked=item.active!false />
            </@formGroup>
            <@formGroup rows=2>
                <@button type='submit' buttonIcon='check' title='#i18n{portal.util.labelValidate}' color='primary' />
                <@aButton href='jsp/admin/plugins/myplugin/ManageItems.jsp?view=manageItems' buttonIcon='times' title='#i18n{portal.util.labelCancel}' />
            </@formGroup>
        </@tform>
    </@pageColumn>
</@pageContainer>
```

### Common v7 → v8 HTML replacements

| v7 (raw HTML / Bootstrap 3-4) | v8 (Freemarker macro) |
|---|----|
| `<div class="panel panel-default">` | `<@box>` or `<@pageContainer>` layout |
| `<table class="table ...">` | `<@table>` |
| `<form method="post" ...>` | `<@tform method='post' ...>` |
| `<div class="form-group"><label>...</label><input ...>` | `<@formGroup labelFor=... labelKey=...><@input .../></@formGroup>` |
| `<input type="text" class="form-control" ...>` | `<@input type='text' .../>` |
| `<textarea class="form-control" ...>` | `<@input type='textarea' ...>` |
| `<select class="form-control" ...>` | `<@select ...>` |
| `<input type="checkbox" ...>` | `<@checkBox .../>` |
| `<button class="btn btn-primary" ...>` | `<@button type='submit' color='primary' .../>` |
| `<a class="btn btn-primary" href="...">` | `<@aButton href='...' color='primary' .../>` |
| `<div class="alert alert-info">` | `<@alert color='info'>` |

### Freemarker best practices

- **Null safety**: always use `${value!}` (with `!`) to handle null values
- **i18n**: use `#i18n{prefix.key}` — reuse existing `portal.util.*` keys when possible (`portal.util.labelActions`, `portal.util.labelModify`, `portal.util.labelDelete`, `portal.util.labelBack`, `portal.util.labelValidate`, `portal.util.labelCancel`)
- **Icons**: use Tabler icon names in `buttonIcon` parameter (`plus`, `edit`, `trash`, `arrow-left`, `check`, `times`)
- **BS5 & Tabler Icons are already loaded** by the admin theme — do NOT add `<link>` or `<script>` for these

---

## 9.2 Front-Office (Skin) Templates

### Wrap with `<@cTpl>`

All skin templates (`templates/skin/**/*.html`) must be wrapped with the `<@cTpl>` macro:

```html
<@cTpl>
  <!-- template content -->
</@cTpl>
```

### Front-office conventions
- Use Bootstrap 5 utility classes (no Bootstrap 3/4)
- No jQuery — vanilla JS only
- No CDN — local assets only (`webapp/js/{pluginName}/`, `webapp/css/{pluginName}/`)

---

## 9.3 JavaScript — No jQuery

Replace all jQuery usage with vanilla JavaScript in both admin and skin templates:

```javascript
// BEFORE (v7) — jQuery
$('#myElement').click(function() { ... });
$.ajax({ url: '...', success: function(data) { ... } });
$('.myClass').hide();

// AFTER (v8) — Vanilla JS
document.querySelector('#myElement').addEventListener('click', () => { ... });
fetch('...').then(response => response.json()).then(data => { ... });
document.querySelector('.myClass').style.display = 'none';
```

Use ES6+: `const`/`let`, arrow functions, template literals, `async`/`await`.

---

## 9.4 Upload Macro Renames (if applicable)

If the plugin uses asynchronous upload macros, rename them:

| v7 Macro | v8 Macro |
|---------|---------|
| `addFileInput` | `addFileBOInput` |
| `addUploadedFilesBox` | `addBOUploadedFilesBox` |
| `addFileInputAndfilesBox` | `addFileBOInputAndfilesBox` |

If the plugin uses jQuery File Upload, replace with Uppy. Use core macros for file uploads:
```html
<@inputDropFiles name=fieldName handler=handler type=type>
    <#nested>
</@inputDropFiles>
```

---

## Verification (MANDATORY before next phase)

1. Grep checks:
   - `grep -rn "class=\"panel" webapp/WEB-INF/templates/admin/` → should return nothing (old Bootstrap panels replaced)
   - `grep -rn "jQuery\|\\$(" webapp/WEB-INF/templates/` → should return nothing (no jQuery)
2. Visual check: verify admin templates use `@pageContainer` > `@pageColumn` > `@pageHeader` layout
3. **No build** — other phases may still have broken references
4. Mark task as completed ONLY when grep checks pass and templates follow v8 macro conventions
