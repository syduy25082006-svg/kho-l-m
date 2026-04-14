<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Hệ Thống Nhập Xuất Kho Hoàn Chỉnh</title>
    <style>
        :root { --bg-purple: #5c5cfc; --accent: #3498db; --danger: #e74c3c; --success: #27ae60; --export: #f39c12; }
        body { font-family: 'Segoe UI', Tahoma, sans-serif; margin: 0; background-color: #f4f6f9; color: #333; }

        .header { background-color: var(--bg-purple); padding: 25px; color: white; text-align: center; }
        .toolbar { background-color: var(--bg-purple); padding: 0 20px 30px; display: flex; gap: 12px; justify-content: center; }

        .btn-excel { 
            background: white; color: var(--bg-purple); border: 2px solid #000; 
            border-radius: 10px; padding: 10px 20px; font-weight: bold; cursor: pointer; 
            display: flex; align-items: center; gap: 8px; 
        }

        /* Form Nhập liệu */
        .input-section {
            max-width: 1050px; margin: -20px auto 20px; background: white; 
            padding: 20px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.1);
            display: grid; grid-template-columns: repeat(auto-fit, minmax(130px, 1fr)); gap: 15px;
        }
        .input-group { display: flex; flex-direction: column; gap: 5px; }
        .input-group label { font-size: 11px; font-weight: bold; color: #888; }
        .input-group input { padding: 10px; border: 1px solid #ddd; border-radius: 6px; outline: none; }

        .btn-group { display: flex; gap: 10px; align-self: flex-end; }
        .btn-action-form { border: none; border-radius: 6px; cursor: pointer; font-weight: bold; height: 40px; color: white; padding: 0 15px; }
        .btn-nhap { background: var(--success); }
        .btn-xuat { background: var(--danger); }

        /* Bảng */
        .main-content { max-width: 1150px; margin: auto; padding: 0 20px; }
        .table-card { background: white; border-radius: 10px; overflow: hidden; box-shadow: 0 2px 10px rgba(0,0,0,0.05); }
        table { width: 100%; border-collapse: collapse; }
        th { background: #fafafa; color: #888; padding: 15px; text-align: left; font-size: 12px; border-bottom: 1px solid #eee; text-transform: uppercase; }
        td { padding: 15px; border-bottom: 1px solid #f2f2f2; font-size: 14px; }

        /* Badge Trạng thái */
        .badge { padding: 4px 8px; border-radius: 4px; font-size: 11px; font-weight: bold; color: white; }
        .badge-nhap { background: var(--success); }
        .badge-xuat { background: var(--danger); }

        .price-text { color: #2c3e50; font-weight: bold; }
        .file-link { color: var(--accent); text-decoration: none; }

        #modal-confirm { 
            display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; 
            background: rgba(0,0,0,0.5); justify-content: center; align-items: center; z-index: 1000; 
        }
        .modal-content { background: white; padding: 30px; border-radius: 15px; width: 320px; text-align: center; }
        #file-input-hidden { display: none; }
    </style>
</head>
<body>

    <div class="header"><h2>HỆ THỐNG QUẢN LÝ NHẬP - XUẤT KHO</h2></div>

    <div class="toolbar">
        <input type="file" id="file-input-hidden">
        <button class="btn-excel" onclick="document.getElementById('file-input-hidden').click()">
            <span>📁</span> Tải Tệp Chứng Từ
        </button>
    </div>

    <div class="input-section">
        <div class="input-group">
            <label>NGƯỜI THỰC HIỆN</label>
            <input type="text" id="user-name" placeholder="Tên cán bộ...">
        </div>
        <div class="input-group">
            <label>SẢN PHẨM</label>
            <input type="text" id="item-name" placeholder="Tên hàng...">
        </div>
        <div class="input-group">
            <label>SỐ LƯỢNG</label>
            <input type="number" id="item-qty" value="1">
        </div>
        <div class="input-group">
            <label>ĐƠN GIÁ (VNĐ)</label>
            <input type="number" id="item-price" placeholder="Giá...">
        </div>
        <div class="btn-group">
            <button class="btn-action-form btn-nhap" onclick="processStock('NHẬP')">NHẬP KHO</button>
            <button class="btn-action-form btn-xuat" onclick="processStock('XUẤT')">XUẤT KHO</button>
        </div>
    </div>

    <div class="main-content">
        <div class="table-card">
            <table>
                <thead>
                    <tr>
                        <th>STT</th>
                        <th>Loại</th>
                        <th>Người Thực Hiện</th>
                        <th>Sản Phẩm</th>
                        <th>Số Lượng</th>
                        <th>Đơn Giá</th>
                        <th>Thành Tiền</th>
                        <th>Chứng Từ</th>
                        <th style="text-align: center;">Thao Tác</th>
                    </tr>
                </thead>
                <tbody id="inventory-data"></tbody>
            </table>
        </div>
    </div>

    <div id="modal-confirm">
        <div class="modal-content">
            <h3>Xác nhận xóa dữ liệu?</h3>
            <div style="display: flex; gap: 10px; justify-content: center; margin-top: 20px;">
                <button onclick="closeModal()" style="padding: 10px 20px; border-radius: 6px; border:none; cursor:pointer;">Hủy</button>
                <button id="btn-delete-confirm" style="padding: 10px 20px; border-radius: 6px; border:none; cursor:pointer; background: var(--danger); color:white;">Xóa</button>
            </div>
        </div>
    </div>

<script>
    const tableBody = document.getElementById('inventory-data');
    let rowToDelete = null;
    let currentFile = null;

    function formatMoney(num) {
        return new Intl.NumberFormat('vi-VN').format(num) + ' ₫';
    }

    document.getElementById('file-input-hidden').onchange = (e) => {
        currentFile = e.target.files[0];
        if(currentFile) alert("Đã chọn file: " + currentFile.name);
    };

    function processStock(type) {
        const name = document.getElementById('user-name').value.trim();
        const item = document.getElementById('item-name').value.trim();
        const qty = parseInt(document.getElementById('item-qty').value);
        const price = parseInt(document.getElementById('item-price').value);

        if (!name || !item || isNaN(qty) || isNaN(price)) {
            alert("Vui lòng nhập đầy đủ thông tin!");
            return;
        }

        const total = qty * price;
        const tr = document.createElement('tr');
        
        let fileHTML = '<span style="color:#ccc">Trống</span>';
        if (currentFile) {
            const url = URL.createObjectURL(currentFile);
            fileHTML = `<a href="${url}" target="_blank" class="file-link" download="${currentFile.name}">📄 Mở file</a>`;
        }

        const badgeClass = type === 'NHẬP' ? 'badge-nhap' : 'badge-xuat';

        tr.innerHTML = `
            <td></td>
            <td><span class="badge ${badgeClass}">${type}</span></td>
            <td><strong>${name}</strong></td>
            <td>${item}</td>
            <td style="text-align:center">${qty}</td>
            <td>${formatMoney(price)}</td>
            <td class="price-text">${formatMoney(total)}</td>
            <td>${fileHTML}</td>
            <td style="text-align: center;">
                <button style="color:var(--accent); background:none; border:none; cursor:pointer;" onclick="editRow(this)">Sửa</button>
                <button style="color:var(--danger); background:none; border:none; cursor:pointer; margin-left:10px;" onclick="askDelete(this)">Xóa</button>
            </td>
        `;

        tableBody.prepend(tr);
        updateSTT();
        
        // Reset Form
        document.getElementById('user-name').value = '';
        document.getElementById('item-name').value = '';
        document.getElementById('item-qty').value = '1';
        document.getElementById('item-price').value = '';
        currentFile = null;
    }

    function updateSTT() {
        const rows = tableBody.querySelectorAll('tr');
        rows.forEach((row, index) => { row.cells[0].innerText = index + 1; });
    }

    function editRow(btn) {
        const row = btn.closest('tr');
        const newQty = prompt("Nhập số lượng mới:", row.cells[4].innerText);
        if (newQty) {
            row.cells[4].innerText = newQty;
            const priceVal = parseInt(row.cells[5].innerText.replace(/[^0-9]/g, ''));
            row.cells[6].innerText = formatMoney(parseInt(newQty) * priceVal);
        }
    }

    function askDelete(btn) {
        rowToDelete = btn.closest('tr');
        document.getElementById('modal-confirm').style.display = 'flex';
    }

    function closeModal() {
        document.getElementById('modal-confirm').style.display = 'none';
        rowToDelete = null;
    }

    document.getElementById('btn-delete-confirm').onclick = () => {
        if (rowToDelete) { rowToDelete.remove(); updateSTT(); closeModal(); }
    };
</script>
</body>
</html>
