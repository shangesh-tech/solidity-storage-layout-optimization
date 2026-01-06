## 1️⃣ First, understand the units (no confusion allowed)

* **Seconds (UNIX timestamp)** → what Solidity gives you
  Example: `1710000000`

* **Milliseconds** → what JavaScript `Date` uses
  Example: `1710000000000`

> JavaScript **always** expects milliseconds.

---

## 2️⃣ Convert **seconds → local date & time**

### Example

```ts
const deadlineSeconds = 1710000000;
```

### Convert properly

```ts
const date = new Date(deadlineSeconds * 1000);
```

### Show local date & time

```ts
date.toString();
```

Example output (depends on your timezone):

```
Sun Mar 10 2024 05:30:00 GMT+0530 (India Standard Time)
```

That’s **local time**, not UTC.

---

## 3️⃣ Convert **milliseconds → local date & time**

### Example

```ts
const deadlineMs = 1710000000000;
```

### Convert

```ts
const date = new Date(deadlineMs);
```

Same result, no multiplication needed.

---

## 4️⃣ Clean formatting (what you actually want in UI)

### Simple readable format

```ts
const date = new Date(deadlineSeconds * 1000);

const formatted = date.toLocaleString();
```

Output:

```
10/3/2024, 5:30:00 am
```

This automatically:

* uses user’s locale
* uses user’s timezone

---

## 5️⃣ Custom format (recommended for production)

```ts
const date = new Date(deadlineSeconds * 1000);

const formatted = date.toLocaleString("en-IN", {
  day: "2-digit",
  month: "short",
  year: "numeric",
  hour: "2-digit",
  minute: "2-digit",
  hour12: true,
});
```

Output:

```
10 Mar 2024, 05:30 AM
```

This is what serious apps show.

---

## 6️⃣ UTC vs Local (don’t ignore this)

### UTC time

```ts
date.toUTCString();
```

Example:

```
Sat, 09 Mar 2024 23:59:59 GMT
```

### Local time

```ts
date.toString();
```

Example:

```
Sun Mar 10 2024 05:30:00 GMT+0530 (IST)
```

**Blockchain time is UTC.
UI time should be LOCAL.**

If you show UTC to users without telling them → bad UX.

---

## 7️⃣ Convert local date back to timestamp (important!)

### From date input → seconds (for smart contract)

```ts
const date = new Date("2024-03-10T05:30");
const seconds = Math.floor(date.getTime() / 1000);
```

### From date input → milliseconds

```ts
const ms = date.getTime();
```

---


