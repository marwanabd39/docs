# Actual PHP Fix (Drop-in) for “Update behaves like Add”

You were right to ask for real code.

Below is a **copy/paste-ready** fix for your exact issue:
- Editing category creates a new category instead of updating.
- Editing item gives “code already exists”.

The root cause is two submit buttons in the same form (`save_*` and `update_*`). Pressing Enter can submit the wrong one.

---

## 1) Replace the two category handlers with one action-based handler

> Remove old blocks:
> - `if(isset($_POST['save_category'])){ ... }`
> - `if(isset($_POST['update_category'])){ ... }`

Use this instead:

```php
/* =====================
   CATEGORY ACTION (SAVE/UPDATE)
===================== */
if (isset($_POST['category_action'])) {
    validate_csrf_token();

    $action = $_POST['category_action']; // save | update
    $name = trim($_POST['cat_name'] ?? '');
    $id = (int)($_POST['cat_id'] ?? 0);

    if ($name === '') {
        $_SESSION['error'] = 'هەڵە: ناوی پۆل پێویستە.';
        header("Location: " . $_SERVER['PHP_SELF']);
        exit();
    }

    if ($action === 'save') {
        // Check if category already exists
        $checkCat = $conn->prepare("SELECT id FROM categories WHERE name = ?");
        $checkCat->bind_param("s", $name);
        $checkCat->execute();
        $checkCat->store_result();

        if ($checkCat->num_rows > 0) {
            $_SESSION['error'] = 'هەڵە: ئەم پۆلە پێشتر تۆمارکراوە!';
        } else {
            $stmt = $conn->prepare("INSERT INTO categories (name) VALUES (?)");
            $stmt->bind_param("s", $name);
            $stmt->execute();
            $_SESSION['success'] = 'پۆل بە سەرکەوتوویی زیادکرا!';
            $stmt->close();
        }
        $checkCat->close();

    } elseif ($action === 'update' && $id > 0) {
        // Check duplicate name excluding current category
        $checkCat = $conn->prepare("SELECT id FROM categories WHERE name = ? AND id != ?");
        $checkCat->bind_param("si", $name, $id);
        $checkCat->execute();
        $checkCat->store_result();

        if ($checkCat->num_rows > 0) {
            $_SESSION['error'] = 'هەڵە: ئەم پۆلە پێشتر تۆمارکراوە!';
        } else {
            $stmt = $conn->prepare("UPDATE categories SET name = ? WHERE id = ?");
            $stmt->bind_param("si", $name, $id);
            $stmt->execute();
            $_SESSION['success'] = 'پۆل بە سەرکەوتوویی نوێکرایەوە!';
            $stmt->close();
        }
        $checkCat->close();

    } else {
        $_SESSION['error'] = 'هەڵە: داواکاری نادروست!';
    }

    header("Location: " . $_SERVER['PHP_SELF']);
    exit();
}
```

---

## 2) Replace the two item handlers with one action-based handler

> Remove old blocks:
> - `if(isset($_POST['save_shoe'])){ ... }`
> - `if(isset($_POST['update_shoe'])){ ... }`

Use this instead:

```php
/* =====================
   ITEM ACTION (SAVE/UPDATE)
===================== */
if (isset($_POST['item_action'])) {
    validate_csrf_token();

    $action = $_POST['item_action']; // save | update

    $id    = (int)($_POST['id'] ?? 0);
    $name  = trim($_POST['name'] ?? '');
    $code  = trim($_POST['code'] ?? '');
    $cat   = (int)($_POST['category'] ?? 0);
    $purchase_location = trim($_POST['purchase_location'] ?? '');
    $carton_quantity = (int)($_POST['carton_quantity'] ?? 0);
    $items_per_carton = (int)($_POST['items_per_carton'] ?? 0);
    $purchase_cost = (float)($_POST['purchase_cost'] ?? 0);
    $wholesale_cost = (float)($_POST['wholesale_cost'] ?? 0);
    $retail_price = (float)($_POST['retail_price'] ?? 0);
    $notes = trim($_POST['notes'] ?? '');

    if ($action === 'save') {
        // Code must be unique
        $checkCode = $conn->prepare("SELECT id FROM additions WHERE code = ?");
        $checkCode->bind_param("s", $code);
        $checkCode->execute();
        $checkCode->store_result();

        if ($checkCode->num_rows > 0) {
            $_SESSION['error'] = 'هەڵە: ئەم کۆدە پێشتر تۆمارکراوە!';
        } else {
            $stmt = $conn->prepare(
                "INSERT INTO additions
                (name,code,category_id,purchase_location,carton_quantity,items_per_carton,purchase_cost,wholesale_cost,retail_price,notes)
                VALUES (?,?,?,?,?,?,?,?,?,?)"
            );
            $stmt->bind_param(
                "ssisiiiddds",
                $name,
                $code,
                $cat,
                $purchase_location,
                $carton_quantity,
                $items_per_carton,
                $purchase_cost,
                $wholesale_cost,
                $retail_price,
                $notes
            );
            $stmt->execute();
            $_SESSION['success'] = 'پێڵاو بە سەرکەوتوویی زیادکرا!';
            $stmt->close();
        }
        $checkCode->close();

    } elseif ($action === 'update' && $id > 0) {
        // Unique code except current record
        $checkCode = $conn->prepare("SELECT id FROM additions WHERE code = ? AND id != ?");
        $checkCode->bind_param("si", $code, $id);
        $checkCode->execute();
        $checkCode->store_result();

        if ($checkCode->num_rows > 0) {
            $_SESSION['error'] = 'هەڵە: ئەم کۆدە پێشتر تۆمارکراوە!';
        } else {
            $sql = "UPDATE additions
                    SET name=?,code=?,category_id=?,purchase_location=?,carton_quantity=?,items_per_carton=?,purchase_cost=?,wholesale_cost=?,retail_price=?,notes=?
                    WHERE id=?";
            $stmt = $conn->prepare($sql);
            $stmt->bind_param(
                "ssisiiidddsi",
                $name,
                $code,
                $cat,
                $purchase_location,
                $carton_quantity,
                $items_per_carton,
                $purchase_cost,
                $wholesale_cost,
                $retail_price,
                $notes,
                $id
            );
            $stmt->execute();
            $_SESSION['success'] = 'پێڵاو بە سەرکەوتوویی نوێکرایەوە!';
            $stmt->close();
        }
        $checkCode->close();

    } else {
        $_SESSION['error'] = 'هەڵە: داواکاری نادروست!';
    }

    header("Location: " . $_SERVER['PHP_SELF']);
    exit();
}
```

---

## 3) Category modal footer: use ONE submit button + hidden action

Replace category footer/buttons with:

```html
<input type="hidden" name="category_action" id="category_action" value="save">

<div class="modal-footer">
    <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">
        <i class="bi bi-x-circle"></i> داخستن
    </button>

    <button type="submit" id="categorySubmitBtn" class="btn btn-success-custom btn-custom">
        <i class="bi bi-check-circle-fill"></i>
        <span id="categorySubmitText">پاشەکەوتکردن</span>
    </button>
</div>
```

---

## 4) Item modal footer: use ONE submit button + hidden action

Replace item footer/buttons with:

```html
<input type="hidden" name="item_action" id="item_action" value="save">

<div class="modal-footer">
    <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">
        <i class="bi bi-x-circle"></i> داخستن
    </button>

    <button type="submit" id="itemSubmitBtn" class="btn btn-success-custom btn-custom">
        <i class="bi bi-check-circle-fill"></i>
        <span id="itemSubmitText">پاشەکەوتکردن</span>
    </button>
</div>
```

---

## 5) JavaScript changes

Replace only the mode-toggle lines in your current JS with this behavior:

```js
function editCat(c){
    const modal = new bootstrap.Modal(document.getElementById('categoryModal'));
    modal.show();

    document.getElementById("catTitle").innerHTML = '<i class="bi bi-pencil-square"></i> پۆل دەستکاریکردن';
    document.getElementById("cat_id").value = c.id;
    document.getElementById("cat_name").value = c.name;

    document.getElementById("category_action").value = "update";
    document.getElementById("categorySubmitText").textContent = "نوێکردنەوە";
}

document.getElementById('categoryModal').addEventListener('hidden.bs.modal', function () {
    document.getElementById("catTitle").innerHTML = '<i class="bi bi-tag-fill"></i> پۆلێن زیادکردن';
    document.getElementById("cat_id").value = "";
    document.getElementById("cat_name").value = "";

    document.getElementById("category_action").value = "save";
    document.getElementById("categorySubmitText").textContent = "پاشەکەوتکردن";
});

function editItem(i){
    const modal = new bootstrap.Modal(document.getElementById('itemModal'));
    modal.show();

    document.getElementById("itemTitle").innerHTML = '<i class="bi bi-pencil-square"></i> پێڵاو دەستکاریکردن';
    document.getElementById("item_id").value = i.id;
    document.getElementById("name").value = i.name;
    document.getElementById("code").value = i.code;
    document.getElementById("category").value = i.category_id;
    document.getElementById("purchase_location").value = i.purchase_location;
    document.getElementById("carton_quantity").value = i.carton_quantity;
    document.getElementById("items_per_carton").value = i.items_per_carton;
    document.getElementById("purchase_cost").value = i.purchase_cost;
    document.getElementById("wholesale_cost").value = i.wholesale_cost;
    document.getElementById("retail_price").value = i.retail_price;
    document.getElementById("notes").value = i.notes;

    document.getElementById("item_action").value = "update";
    document.getElementById("itemSubmitText").textContent = "نوێکردنەوە";
}

document.getElementById('itemModal').addEventListener('hidden.bs.modal', function () {
    document.getElementById("itemTitle").innerHTML = '<i class="bi bi-box-seam-fill"></i> پێڵاو زیادکردن';
    document.getElementById("item_id").value = "";
    document.getElementById("name").value = "";
    document.getElementById("code").value = "";
    document.getElementById("category").value = "";
    document.getElementById("purchase_location").value = "";
    document.getElementById("carton_quantity").value = "";
    document.getElementById("items_per_carton").value = "";
    document.getElementById("purchase_cost").value = "";
    document.getElementById("wholesale_cost").value = "";
    document.getElementById("retail_price").value = "";
    document.getElementById("notes").value = "";

    document.getElementById("item_action").value = "save";
    document.getElementById("itemSubmitText").textContent = "پاشەکەوتکردن";
});
```

---

## Important note
This repository does not contain your runtime PHP app files (`*.php`), so I cannot patch your live file directly here. This file now contains the **actual PHP/HTML/JS code** you can paste into your app file.
