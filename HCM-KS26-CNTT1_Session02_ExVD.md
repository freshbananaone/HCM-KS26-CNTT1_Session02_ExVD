# BÀI TẬP: THIẾT KẾ GIẢI PHÁP PROMPTING ỨNG DỤNG THỰC TẾ

**Chủ đề:** Cấu trúc CREATE, Few-shot và ép định dạng JSON/Table  
**Tình huống chọn:** Tự động phân loại phản hồi khách hàng của cửa hàng MartX.

## Bước 1. Thiết kế giải pháp (Input / Output)

### 1. Bài toán

Mỗi ngày cửa hàng nhận nhiều phản hồi về sản phẩm và đơn hàng. Nếu nhân viên đọc và phân loại thủ công thì mất thời gian, dễ bỏ sót các phản hồi cần hỗ trợ. Em đề xuất dùng AI để phân loại phản hồi tự động và đánh dấu dữ liệu bị thiếu hoặc sai.

### 2. Input (dữ liệu đầu vào)

Mỗi phản hồi được gửi dưới dạng JSON, gồm:

| Trường | Ý nghĩa | Ví dụ |
|---|---|---|
| `ma_phan_hoi` | Mã để phân biệt từng phản hồi | `PH01` |
| `ma_don` | Mã đơn hàng, có dạng `MTX-` và 4 chữ số | `MTX-9988` |
| `noi_dung` | Nội dung khách hàng phản hồi | `Rau tươi, giao nhanh` |

**Trường hợp cần xử lý:** thiếu mã đơn, mã đơn sai định dạng, nội dung trống hoặc nội dung không đủ rõ để phân loại.

### 3. Output (dữ liệu đầu ra)

AI trả về **2 phần**:

- **JSON chuẩn:** gồm danh sách `ket_qua`, mỗi phần tử có `ma_phan_hoi`, `ma_don`, `nhan`, `trang_thai`, `ghi_chu`.
- **Bảng Markdown:** có đúng 5 cột tương ứng để nhân viên dễ xem.

Quy ước:

- `nhan`: chỉ nhận `TICH_CUC`, `TIEU_CUC`, `CAN_HO_TRO` hoặc `KHONG_DU_LIEU`.
- `trang_thai`: `HOP_LE` hoặc `CAN_KIEM_TRA`.
- `ma_don`: trả về `null` nếu bị thiếu hoặc sai định dạng, không tự điền mã khác.
- `ghi_chu`: nêu ngắn gọn lỗi hoặc lý do cần hỗ trợ; nếu không có thì ghi `Không`.

## Bước 2. Viết prompt hoàn chỉnh theo CREATE và chạy thử

### 1. Prompt sử dụng

~~~text
[C - CONTEXT / NGỮ CẢNH]
Bạn là trợ lý AI hỗ trợ bộ phận chăm sóc khách hàng của cửa hàng MartX.
Cửa hàng muốn tự động phân loại phản hồi để nhân viên ưu tiên xử lý đúng việc.

[R - REQUEST / YÊU CẦU]
Đọc danh sách phản hồi khách hàng dạng JSON, phân loại từng phản hồi và kiểm tra dữ liệu lỗi.

[E - EXPLANATION / GIẢI THÍCH]
Việc phân loại cần chính xác, thống nhất và không tự bịa dữ liệu.
Quy tắc phân loại theo thứ tự ưu tiên:
1. Nếu noi_dung trống, không phải chuỗi hoặc không đủ rõ để xác định: KHONG_DU_LIEU.
2. Nếu khách yêu cầu đổi/trả/hoàn tiền/hỗ trợ, hoặc báo giao thiếu, giao sai, hàng hỏng cần xử lý: CAN_HO_TRO.
3. Nếu khách khen và không nêu vấn đề cần xử lý: TICH_CUC.
4. Nếu khách chê/phàn nàn nhưng không yêu cầu xử lý cụ thể: TIEU_CUC.

[A - ACTION / HÀNH ĐỘNG]
- Đọc lần lượt từng phần tử trong danh sách.
- Kiểm tra ma_don đúng mẫu MTX- + đúng 4 chữ số; nếu sai hoặc trống, gán null.
- Phân loại nhan theo quy tắc trên.
- Gán trang_thai = CAN_KIEM_TRA nếu ma_don sai/thiếu, noi_dung trống hoặc không rõ;
  ngược lại gán HOP_LE.
- Ghi ghi_chu ngắn gọn, nêu đủ các lỗi nếu có.
- Xuất kết quả thành 2 phần: (1) khối JSON hợp lệ; (2) bảng Markdown 5 cột.

[T - TONE / GIỌNG ĐIỆU]
Ngắn gọn, rõ ràng, dễ hiểu, phù hợp cho nhân viên cửa hàng.

[E - EXTRAS / RÀNG BUỘC]
- Không đoán mã đơn, không tự tạo phản hồi hoặc thông tin khách hàng.
- Giữ nguyên thứ tự và số lượng các phản hồi đầu vào.
- Chỉ dùng 4 nhãn và 2 trạng thái đã quy định.
- Nếu không đủ căn cứ phân loại, dùng KHONG_DU_LIEU thay vì đoán.
- JSON phải hợp lệ: dùng dấu ngoặc kép, null (không phải "null"), không có dấu phẩy thừa.
- Bảng Markdown phải khớp 100% với dữ liệu JSON.

[FEW-SHOT / VÍ DỤ MẪU]
Ví dụ 1:
Input: {"ma_phan_hoi":"VD01","ma_don":"MTX-1001","noi_dung":"Rau rất tươi, giao hàng nhanh."}
Output: {"ma_phan_hoi":"VD01","ma_don":"MTX-1001","nhan":"TICH_CUC","trang_thai":"HOP_LE","ghi_chu":"Không"}

Ví dụ 2:
Input: {"ma_phan_hoi":"VD02","ma_don":"MTX-1002","noi_dung":"Hàng bị dập, tôi muốn đổi hàng."}
Output: {"ma_phan_hoi":"VD02","ma_don":"MTX-1002","nhan":"CAN_HO_TRO","trang_thai":"HOP_LE","ghi_chu":"Khách yêu cầu đổi hàng"}

Ví dụ 3:
Input: {"ma_phan_hoi":"VD03","ma_don":"sai-ma","noi_dung":"Giá hơi cao, tôi không hài lòng."}
Output: {"ma_phan_hoi":"VD03","ma_don":null,"nhan":"TIEU_CUC","trang_thai":"CAN_KIEM_TRA","ghi_chu":"Mã đơn hàng sai định dạng"}

[DỮ LIỆU CẦN XỬ LÝ]
[
  {"ma_phan_hoi":"PH01","ma_don":"MTX-9988","noi_dung":"Rau rất tươi, giao nhanh, sẽ mua lại."},
  {"ma_phan_hoi":"PH02","ma_don":"MTX-9989","noi_dung":"Shop giao thiếu 1 kg cà chua, nhờ hỗ trợ giúp."},
  {"ma_phan_hoi":"PH03","ma_don":"","noi_dung":"Giá đắt, tôi không hài lòng."},
  {"ma_phan_hoi":"PH04","ma_don":"MTX-ABCD","noi_dung":""},
  {"ma_phan_hoi":"PH05","ma_don":"MTX-9990","noi_dung":"Đã nhận được hàng."}
]
~~~

### 2. Kết quả AI trả về khi chạy thử

**Phần A – JSON:**

~~~json
{
  "ket_qua": [
    {
      "ma_phan_hoi": "PH01",
      "ma_don": "MTX-9988",
      "nhan": "TICH_CUC",
      "trang_thai": "HOP_LE",
      "ghi_chu": "Không"
    },
    {
      "ma_phan_hoi": "PH02",
      "ma_don": "MTX-9989",
      "nhan": "CAN_HO_TRO",
      "trang_thai": "HOP_LE",
      "ghi_chu": "Khách báo giao thiếu hàng và yêu cầu hỗ trợ"
    },
    {
      "ma_phan_hoi": "PH03",
      "ma_don": null,
      "nhan": "TIEU_CUC",
      "trang_thai": "CAN_KIEM_TRA",
      "ghi_chu": "Thiếu mã đơn hàng"
    },
    {
      "ma_phan_hoi": "PH04",
      "ma_don": null,
      "nhan": "KHONG_DU_LIEU",
      "trang_thai": "CAN_KIEM_TRA",
      "ghi_chu": "Mã đơn hàng sai định dạng; thiếu nội dung phản hồi"
    },
    {
      "ma_phan_hoi": "PH05",
      "ma_don": "MTX-9990",
      "nhan": "KHONG_DU_LIEU",
      "trang_thai": "CAN_KIEM_TRA",
      "ghi_chu": "Nội dung chưa rõ tích cực, tiêu cực hay cần hỗ trợ"
    }
  ]
}
~~~

