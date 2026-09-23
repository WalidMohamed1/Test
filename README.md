package readertojson;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.math.BigDecimal;
import java.math.MathContext;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Base64;
import java.util.Calendar;
import java.util.GregorianCalendar;
import java.util.HashMap;
import java.util.HashSet;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Scanner;
import java.util.Set;
import java.util.TreeMap;
import java.util.logging.Level;
import java.util.logging.Logger;
import java.util.zip.ZipEntry;
import java.util.zip.ZipInputStream;
import javax.xml.parsers.DocumentBuilder;
import javax.xml.parsers.DocumentBuilderFactory;
import org.w3c.dom.Document;
import org.w3c.dom.Element;
import org.w3c.dom.Node;
import org.w3c.dom.NodeList;

/**
 * Reads a base64-encoded .xlsx file and returns its first sheet as a JSON array.
 * The first row is used as keys; every following row becomes one object.
 * Uses only the JDK (no Apache POI / xmlbeans), Java 8 compatible.
 */
public class ReaderToJson {

    private static final Logger logger = Logger.getLogger("com.ibm.bpm.custom.ReadExcel");

    private static final String NS_MAIN = "http://schemas.openxmlformats.org/spreadsheetml/2006/main";
    private static final String NS_REL  = "http://schemas.openxmlformats.org/officeDocument/2006/relationships";
    private static final String NS_PKG  = "http://schemas.openxmlformats.org/package/2006/relationships";

    // Local testing only: pass a file path or base64 string as argument, or enter it when asked
    public static void main(String[] args) throws Exception {
        String input;
        if (args.length > 0) {
            input = args[0];
        } else {
            Scanner scanner = new Scanner(System.in);
            System.out.print("Enter Excel file path OR base64: ");
            input = scanner.nextLine().trim();
        }

        String base64 = isFile(input)
                ? Base64.getEncoder().encodeToString(Files.readAllBytes(Paths.get(input)))
                : input;

        System.out.println(new ReaderToJson().read(base64));
    }

    private static boolean isFile(String s) {
        try {
            return s.length() < 1000 && Files.exists(Paths.get(s));
        } catch (Exception e) {
            return false;
        }
    }

    public String read(String base64ExcelData) {
        if (base64ExcelData == null || base64ExcelData.trim().isEmpty()) {
            throw new RuntimeException("ReadExcel(read) - The Excel data passed is either missing or bad.");
        }
        try {
            byte[] data = Base64.getMimeDecoder().decode(base64ExcelData.trim());
            if (data.length < 2 || data[0] != 'P' || data[1] != 'K') {
                throw new RuntimeException("ReadExcel(read) - Only .xlsx files are supported.");
            }

            Map<String, byte[]> files = unzip(data);
            DocumentBuilder db = newBuilder();

            List<String> sharedStrings = readSharedStrings(files, db);
            Set<Integer> dateStyles = readDateStyles(files, db);
            Document sheet = parse(files, findFirstSheetPath(files, db), db);

            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSSZ");
            List<TreeMap<Integer, String>> rows = readRows(sheet, sharedStrings, dateStyles, sdf);

            List<LinkedHashMap<String, String>> records = new ArrayList<>();
            if (rows.isEmpty()) {
                return buildJsonArray(records);
            }

            TreeMap<Integer, String> headers = rows.get(0);
            for (int i = 1; i < rows.size(); i++) {
                TreeMap<Integer, String> row = rows.get(i);
                LinkedHashMap<String, String> record = new LinkedHashMap<>();
                boolean hasData = false;

                for (Map.Entry<Integer, String> h : headers.entrySet()) {
                    if (h.getValue().isEmpty()) continue;
                    String value = row.containsKey(h.getKey()) ? row.get(h.getKey()) : "";
                    if (!value.isEmpty()) hasData = true;
                    record.put(h.getValue(), value);
                }
                if (hasData) records.add(record);
            }
            return buildJsonArray(records);

        } catch (RuntimeException e) {
            throw e;
        } catch (Exception e) {
            logger.logp(Level.SEVERE, "com.ibm.bpm.custom.ReadExcel", "read", "error reading Excel file", e);
            throw new RuntimeException("ReadExcel(read) - error reading Excel file: " + e.getMessage(), e);
        }
    }

    // ---------- xlsx parsing ----------

    private Map<String, byte[]> unzip(byte[] data) throws IOException {
        Map<String, byte[]> files = new HashMap<>();
        try (ZipInputStream zis = new ZipInputStream(new ByteArrayInputStream(data))) {
            ZipEntry entry;
            byte[] buffer = new byte[8192];
            while ((entry = zis.getNextEntry()) != null) {
                if (entry.isDirectory()) continue;
                ByteArrayOutputStream out = new ByteArrayOutputStream();
                int n;
                while ((n = zis.read(buffer)) > 0) out.write(buffer, 0, n);
                files.put(entry.getName(), out.toByteArray());
            }
        }
        return files;
    }

    private DocumentBuilder newBuilder() throws Exception {
        DocumentBuilderFactory f = DocumentBuilderFactory.newInstance();
        f.setNamespaceAware(true);
        f.setExpandEntityReferences(false);
        try {
            f.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
        } catch (Exception ignore) {
            // feature not supported by this parser, continue
        }
        return f.newDocumentBuilder();
    }

    private Document parse(Map<String, byte[]> files, String name, DocumentBuilder db) throws Exception {
        byte[] bytes = files.get(name);
        if (bytes == null) throw new IOException("Missing part in xlsx: " + name);
        return db.parse(new ByteArrayInputStream(bytes));
    }

    private String findFirstSheetPath(Map<String, byte[]> files, DocumentBuilder db) throws Exception {
        Document wb = parse(files, "xl/workbook.xml", db);
        NodeList sheets = wb.getElementsByTagNameNS(NS_MAIN, "sheet");
        if (sheets.getLength() > 0 && files.containsKey("xl/_rels/workbook.xml.rels")) {
            String relId = ((Element) sheets.item(0)).getAttributeNS(NS_REL, "id");
            Document rels = parse(files, "xl/_rels/workbook.xml.rels", db);
            NodeList list = rels.getElementsByTagNameNS(NS_PKG, "Relationship");
            for (int i = 0; i < list.getLength(); i++) {
                Element rel = (Element) list.item(i);
                if (relId.equals(rel.getAttribute("Id"))) {
                    String target = rel.getAttribute("Target");
                    return target.startsWith("/") ? target.substring(1) : "xl/" + target;
                }
            }
        }
        return "xl/worksheets/sheet1.xml";
    }

    private List<String> readSharedStrings(Map<String, byte[]> files, DocumentBuilder db) throws Exception {
        List<String> list = new ArrayList<>();
        if (!files.containsKey("xl/sharedStrings.xml")) return list;
        Document doc = parse(files, "xl/sharedStrings.xml", db);
        NodeList items = doc.getElementsByTagNameNS(NS_MAIN, "si");
        for (int i = 0; i < items.getLength(); i++) {
            list.add(textOf((Element) items.item(i)));
        }
        return list;
    }

    private String textOf(Element parent) {
        StringBuilder sb = new StringBuilder();
        NodeList ts = parent.getElementsByTagNameNS(NS_MAIN, "t");
        for (int i = 0; i < ts.getLength(); i++) {
            Node t = ts.item(i);
            if (!"rPh".equals(t.getParentNode().getLocalName())) {
                sb.append(t.getTextContent());
            }
        }
        return sb.toString();
    }

    private Set<Integer> readDateStyles(Map<String, byte[]> files, DocumentBuilder db) throws Exception {
        Set<Integer> result = new HashSet<>();
        if (!files.containsKey("xl/styles.xml")) return result;
        Document doc = parse(files, "xl/styles.xml", db);

        Map<Integer, String> customFormats = new HashMap<>();
        NodeList fmts = doc.getElementsByTagNameNS(NS_MAIN, "numFmt");
        for (int i = 0; i < fmts.getLength(); i++) {
            Element f = (Element) fmts.item(i);
            customFormats.put(Integer.parseInt(f.getAttribute("numFmtId")), f.getAttribute("formatCode"));
        }

        NodeList cellXfs = doc.getElementsByTagNameNS(NS_MAIN, "cellXfs");
        if (cellXfs.getLength() == 0) return result;

        int index = 0;
        for (Node n = cellXfs.item(0).getFirstChild(); n != null; n = n.getNextSibling()) {
            if (n.getNodeType() != Node.ELEMENT_NODE || !"xf".equals(n.getLocalName())) continue;
            String idAttr = ((Element) n).getAttribute("numFmtId");
            int fmtId = idAttr.isEmpty() ? 0 : Integer.parseInt(idAttr);
            if (isDateFormat(fmtId, customFormats.get(fmtId))) result.add(index);
            index++;
        }
        return result;
    }

    private boolean isDateFormat(int id, String code) {
        if (code == null) {
            return (id >= 14 && id <= 22) || (id >= 27 && id <= 36)
                || (id >= 45 && id <= 47) || (id >= 50 && id <= 58);
        }
        String cleaned = code.replaceAll("\"[^\"]*\"", "")
                             .replaceAll("\\[[^\\]]*\\]", "")
                             .replaceAll("\\\\.", "")
                             .toLowerCase();
        return cleaned.matches(".*[ymdhs].*");
    }

    private List<TreeMap<Integer, String>> readRows(Document sheet, List<String> sharedStrings,
                                                     Set<Integer> dateStyles, SimpleDateFormat sdf) {
        List<TreeMap<Integer, String>> rows = new ArrayList<>();
        NodeList rowNodes = sheet.getElementsByTagNameNS(NS_MAIN, "row");
        for (int i = 0; i < rowNodes.getLength(); i++) {
            TreeMap<Integer, String> row = new TreeMap<>();
            NodeList cells = ((Element) rowNodes.item(i)).getElementsByTagNameNS(NS_MAIN, "c");
            int nextCol = 0;
            for (int j = 0; j < cells.getLength(); j++) {
                Element c = (Element) cells.item(j);
                String ref = c.getAttribute("r");
                int col = ref.isEmpty() ? nextCol : columnIndex(ref);
                nextCol = col + 1;
                row.put(col, cellValue(c, sharedStrings, dateStyles, sdf));
            }
            rows.add(row);
        }
        return rows;
    }

    private int columnIndex(String ref) {
        int col = 0;
        for (int i = 0; i < ref.length(); i++) {
            char ch = ref.charAt(i);
            if (!Character.isLetter(ch)) break;
            col = col * 26 + (Character.toUpperCase(ch) - 'A' + 1);
        }
        return col - 1;
    }

    private String cellValue(Element c, List<String> sharedStrings,
                             Set<Integer> dateStyles, SimpleDateFormat sdf) {
        String type = c.getAttribute("t");

        if ("inlineStr".equals(type)) {
            NodeList is = c.getElementsByTagNameNS(NS_MAIN, "is");
            return is.getLength() > 0 ? textOf((Element) is.item(0)) : "";
        }

        NodeList vs = c.getElementsByTagNameNS(NS_MAIN, "v");
        if (vs.getLength() == 0) return "";
        String v = vs.item(0).getTextContent();

        if ("s".equals(type)) {
            int idx = Integer.parseInt(v.trim());
            return idx < sharedStrings.size() ? sharedStrings.get(idx) : "";
        }
        if ("b".equals(type)) return "1".equals(v.trim()) ? "true" : "false";
        if ("str".equals(type) || "e".equals(type)) return v;

        String style = c.getAttribute("s");
        if (!style.isEmpty() && dateStyles.contains(Integer.parseInt(style))) {
            try {
                return formatDate(Double.parseDouble(v), sdf);
            } catch (NumberFormatException e) {
                return v;
            }
        }
        return formatNumber(v);
    }

    private String formatDate(double serial, SimpleDateFormat sdf) {
        int wholeDays = (int) Math.floor(serial);
        long millis = Math.round((serial - wholeDays) * 86400000L);
        Calendar cal = new GregorianCalendar(1899, Calendar.DECEMBER, wholeDays < 61 ? 31 : 30);
        cal.add(Calendar.DAY_OF_MONTH, wholeDays);
        cal.add(Calendar.MILLISECOND, (int) millis);
        return sdf.format(cal.getTime());
    }

    private String formatNumber(String v) {
        try {
            return new BigDecimal(v.trim()).round(new MathContext(15))
                    .stripTrailingZeros().toPlainString();
        } catch (NumberFormatException e) {
            return v;
        }
    }

    // ---------- JSON output ----------

    private String buildJsonArray(List<LinkedHashMap<String, String>> records) {
        StringBuilder json = new StringBuilder("[\n");
        for (int i = 0; i < records.size(); i++) {
            LinkedHashMap<String, String> record = records.get(i);
            json.append("    {\n");
            int j = 0;
            for (Map.Entry<String, String> entry : record.entrySet()) {
                json.append("        \"").append(escapeJson(entry.getKey()))
                    .append("\": \"").append(escapeJson(entry.getValue())).append("\"");
                if (j < record.size() - 1) json.append(",");
                json.append("\n");
                j++;
            }
            json.append("    }");
            if (i < records.size() - 1) json.append(",");
            json.append("\n");
        }
        return json.append("]").toString();
    }

    private String escapeJson(String value) {
        if (value == null) return "";
        return value.replace("\\", "\\\\")
                    .replace("\"", "\\\"")
                    .replace("\n", "\\n")
                    .replace("\r", "\\r")
                    .replace("\t", "\\t");
    }
}

