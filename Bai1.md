## Bài 1: Phân tích & Lựa chọn

- **Đáp án lựa chọn:** B

### Vì sao chọn phương án B
- Phương án B cung cấp đầy đủ cấu trúc 4 phần kỹ thuật:
  - Input: định nghĩa chuỗi XML và các trường `<id>`, `<amount>`, `<status>`.
  - Processing: yêu cầu dùng XML parser tích hợp Java, kiểm duyệt biên, ném lỗi nếu vi phạm.
  - Output: trả về `List<TransactionDTO>`.
  - Language: rõ ràng là Java 17 và dùng record `TransactionDTO`.
- B còn yêu cầu Dry-run CoT, tức AI phải mô tả thuật toán và xử lý lỗi trước khi sinh code. Điều này giúp giảm nguy cơ AI viết code sai hoặc bỏ qua kiểm tra XML.
- B nêu rõ ngoại lệ cần xử lý: `InvalidTransactionException`, `XMLStreamException` hoặc `ParserConfigurationException`, nên prompt buộc AI quan tâm đến parsing error và boundary validation.

### Nhược điểm của phương án còn lại
- Phương án A:
  - Quá ngắn và quá chung chung.
  - Không xác định rõ cấu trúc XML, không yêu cầu validate `amount > 0` và `id` không rỗng một cách cụ thể.
  - Dễ khiến AI bỏ qua xử lý lỗi phân tích XML hoặc chỉ trả về `null`/list rỗng.
- Phương án C:
  - Sai yêu cầu bài toán: dùng Spring XML Reader, stream song song, lưu thẳng vào database.
  - Không phù hợp vì đề bài chỉ cần ánh xạ XML thành `TransactionDTO` và ném ngoại lệ khi dữ liệu không hợp lệ.
  - Còn mang thêm dependency/thiết kế không cần thiết, gây rối khi AI sinh code.

### Mã nguồn Java hoàn chỉnh
```java
import org.w3c.dom.Document;
import org.w3c.dom.Element;
import org.w3c.dom.Node;
import org.w3c.dom.NodeList;
import org.xml.sax.InputSource;
import org.xml.sax.SAXException;

import javax.xml.parsers.DocumentBuilder;
import javax.xml.parsers.DocumentBuilderFactory;
import javax.xml.parsers.ParserConfigurationException;
import java.io.IOException;
import java.io.StringReader;
import java.util.ArrayList;
import java.util.List;

public record TransactionDTO(String id, double amount, String status) {}

public class InvalidTransactionException extends RuntimeException {
    public InvalidTransactionException(String message) {
        super(message);
    }

    public InvalidTransactionException(String message, Throwable cause) {
        super(message, cause);
    }
}

public class TransactionParser {

    public static List<TransactionDTO> parseTransactions(String xmlContent) {
        try {
            DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
            DocumentBuilder builder = factory.newDocumentBuilder();
            Document document = builder.parse(new InputSource(new StringReader(xmlContent)));
            document.getDocumentElement().normalize();

            NodeList transactionNodes = document.getElementsByTagName("transaction");
            List<TransactionDTO> transactions = new ArrayList<>();

            for (int i = 0; i < transactionNodes.getLength(); i++) {
                Node node = transactionNodes.item(i);
                if (node.getNodeType() != Node.ELEMENT_NODE) {
                    continue;
                }

                Element transactionElement = (Element) node;
                String id = getTagValue(transactionElement, "id");
                String amountText = getTagValue(transactionElement, "amount");
                String status = getTagValue(transactionElement, "status");

                validateTransaction(id, amountText);
                double amount = parseAmount(amountText);

                transactions.add(new TransactionDTO(id, amount, status));
            }

            return transactions;
        } catch (ParserConfigurationException | SAXException | IOException ex) {
            throw new InvalidTransactionException("Lỗi phân tích XML: " + ex.getMessage(), ex);
        }
    }

    private static String getTagValue(Element parent, String tagName) {
        NodeList nodes = parent.getElementsByTagName(tagName);
        if (nodes.getLength() == 0) {
            return "";
        }
        return nodes.item(0).getTextContent().trim();
    }

    private static double parseAmount(String amountText) {
        try {
            return Double.parseDouble(amountText);
        } catch (NumberFormatException ex) {
            throw new InvalidTransactionException("Số tiền không hợp lệ: " + amountText, ex);
        }
    }

    private static void validateTransaction(String id, String amountText) {
        if (id == null || id.isBlank()) {
            throw new InvalidTransactionException("Mã giao dịch id không được để trống.");
        }
        double amount = parseAmount(amountText);
        if (amount <= 0) {
            throw new InvalidTransactionException("Số tiền amount phải lớn hơn 0.");
        }
    }
}
```
