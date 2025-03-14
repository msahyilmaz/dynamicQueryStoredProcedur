 
# Dinamik T-SQL Sorgusu ve Stored Procedure Kullanımı

## Açıklama
Bu örnek, dinamik bir T-SQL sorgusunun nasıl oluşturulacağı ve bir **Stored Procedure** içinde nasıl kullanılacağı konusunda yardımcı olacak bir örnek sunmaktadır. Genellikle, birçok projede listeleme işlemleri sırasında **arama**, **sayfalama**, **filtreleme** gibi ihtiyaçlar ortaya çıkar. 

Örneğin, bir müşterinin verilerini listelerken, kullanıcı iki veya daha fazla müşterinin verilerini aynı anda listelemek istediğinde `=` operatörü yerine `IN` operatörünü kullanmak gerekir. Performans açısından farklılıklar olabilir, ancak burada amacımız bu durumda çözüm sağlayacak bir kod parçası sunmaktır. 

Bu örnekte dinamik bir sorgu ile:
- Sayfalama nasıl yapılır,
- Birden fazla tabloda nasıl arama yapılır,
- Birden fazla ID ile nasıl filtreleme yapılır,
- Sıralama nasıl yapılır, gösterilmektedir.
## Örnek UI ekranı

![Filtreleme Ekranı](https://example.com/örnek_resim.png)

## Stored Procedure SQL

```sql
CREATE PROCEDURE sp_dynamicQueryStoredProcedure
    @MID NVARCHAR(MAX) = NULL,
    @ISLEM_ID NVARCHAR(MAX) = NULL,
    @SEARCH NVARCHAR(MAX) = NULL,
    @TARIH_BAS NVARCHAR(MAX) = NULL,
    @TARIH_BIT NVARCHAR(MAX) = NULL,
    @Page INT = 0,
    @ItemCount INT = 10
AS
BEGIN
 SET NOCOUNT ON;

    DECLARE @SQL NVARCHAR(MAX);
    
    SET @SQL = '
    WITH CTE AS (
        SELECT 
            A.ID,
            M.KURUM_AD AS MUSTERI_AD,
            MI.AD AS ISLEM_AD,
            A.TUTAR, 
            MFT.AD AS FIYAT_TUR_AD,
            MFT.ICON AS FIYAT_TUR_ICON,
            A.ACIKLAMA,
            A.FATURA,
            A.TARIH,
            A.FIS_ID,
            A.ODEME_ID,
            COUNT(*) OVER () AS TotalCount,
            CAST(CEILING(COUNT(*) OVER () * 1.0 / @ItemCount) AS INT) AS TotalPages
        FROM [dbo].[MUHASEBE_HAREKET] AS A
        LEFT JOIN MUSTERILER AS M ON M.ID = A.MID
        LEFT JOIN MUHASEBE_ISLEM AS MI ON MI.ID = A.ISLEM_ID
        LEFT JOIN MUHASEBE_FIYAT_TUR AS MFT ON MFT.ID = A.FIYAT_TUR_ID
        WHERE A.STATUSCODE = 1 ';

    -- Dinamik filtreler ekliyoruz.
    IF @MID IS NOT NULL
        SET @SQL = @SQL + ' AND A.MID IN (SELECT value FROM STRING_SPLIT(@MID, '',''))';

    IF @ISLEM_ID IS NOT NULL
        SET @SQL = @SQL + ' AND A.ISLEM_ID IN (SELECT value FROM STRING_SPLIT(@ISLEM_ID, '',''))';

    -- Search parametresi null değilse arama kriterlerni gelen kelimeyi ekle.
    IF @SEARCH IS NOT NULL
        SET @SQL = @SQL + ' AND (
            A.ACIKLAMA LIKE ''%' + @SEARCH + '%'' OR 
            A.FATURA LIKE ''%' + @SEARCH + '%'' OR 
            M.KURUM_AD LIKE ''%' + @SEARCH + '%'' OR 
            MI.AD LIKE ''%' + @SEARCH + '%'' OR 
            MFT.AD LIKE ''%' + @SEARCH + '%''
        )';

    -- Tarih filtreleme
    IF @TARIH_BAS IS NOT NULL AND @TARIH_BIT IS NOT NULL
        SET @SQL = @SQL + ' AND A.TARIH BETWEEN TRY_CONVERT(DATETIME, @TARIH_BAS, 120) AND TRY_CONVERT(DATETIME, @TARIH_BIT, 120)';

    -- Sayfalama işlemi
    SET @SQL = @SQL + '
    )
    SELECT * FROM CTE
    ORDER BY ID DESC
    OFFSET @Page * @ItemCount ROWS
    FETCH NEXT @ItemCount ROWS ONLY;';

    -- Dinamik SQL çalıştırma
    EXEC sp_executesql @SQL,
        N'@MID NVARCHAR(MAX), @ISLEM_ID NVARCHAR(MAX), @SEARCH NVARCHAR(MAX), @TARIH_BAS NVARCHAR(MAX), @TARIH_BIT NVARCHAR(MAX), @Page INT, @ItemCount INT',
        @MID, @ISLEM_ID, @SEARCH, @TARIH_BAS, @TARIH_BIT, @Page, @ItemCount;
END;
```

## Kullanım SQL

```sql
EXEC sp_dynamicQueryStoredProcedure
    @MID = '1,2,3',
    @ISLEM_ID = '5,6',
    @SEARCH = 'Fatura123',
    @TARIH_BAS = '2024-01-01 00:00:00',
    @TARIH_BIT = '2024-12-31 23:59:59',
    @Page = 0,
    @ItemCount = 10;
```

## Kullanım C#

```csharp
public class DynamicQueryStoredProcedureItem
{
    public long ID { get; set; }
    public string MUSTERI_AD { get; set; }
    public string ISLEM_AD { get; set; }
    public decimal TUTAR { get; set; }
    public string FIYAT_TUR_AD { get; set; }
    public string FIYAT_TUR_ICON { get; set; }
    public string ACIKLAMA { get; set; }
    public string FATURA { get; set; }
    public DateTime TARIH { get; set; }
    public long? FIS_ID { get; set; }
    public long? ODEME_ID { get; set; }
    public int TotalCount { get; set; } // Toplam sayfa sayısını hesaplamak için
    public int TotalPages { get; set; } // Toplam sayfa sayısı
}

// Parametreler
SqlParameter[] pDynamicQueryStoredProcedure = {
    new SqlParameter("@MID", string.IsNullOrEmpty(request.MID) ? (object)DBNull.Value : request.MID),
    new SqlParameter("@ISLEM_ID", string.IsNullOrEmpty(request.ISLEM_ID) ? (object)DBNull.Value : request.ISLEM_ID),
    new SqlParameter("@SEARCH", string.IsNullOrEmpty(request.SEARCH) ? (object)DBNull.Value : request.SEARCH),
    new SqlParameter("@TARIH_BAS", string.IsNullOrEmpty(request.TARIH_BAS) ? (object)DBNull.Value : request.TARIH_BAS),
    new SqlParameter("@TARIH_BIT", string.IsNullOrEmpty(request.TARIH_BIT) ? (object)DBNull.Value : request.TARIH_BIT),
    new SqlParameter("@Page", request.Page),
    new SqlParameter("@ItemCount", request.ItemCount == 0 ? 10 : request.ItemCount)
};

// Sorguyu çalıştırma
List<DynamicQueryStoredProcedureItem> qDynamicQueryStoredProcedure = _context.SqlQuery<DynamicQueryStoredProcedureItem>(
    "[dbo].[sp_dynamicQueryStoredProcedure] @MID, @ISLEM_ID, @SEARCH, @TARIH_BAS, @TARIH_BIT", pDynamicQueryStoredProcedure
).ToList();
```

## Parametreler

| Parametre      | Tipi          | Varsayılan Değer | Açıklama |
|----------------|---------------|------------------|----------|
| `@MID`         | NVARCHAR(MAX) | NULL             | Birden fazla müşteri ID'sini virgülle ayırarak filtrelemek için kullanılır. |
| `@ISLEM_ID`    | NVARCHAR(MAX) | NULL             | Birden fazla işlem ID'sini virgülle ayırarak filtrelemek için kullanılır. |
| `@SEARCH`      | NVARCHAR(MAX) | NULL             | Açıklama, fatura, müşteri adı, işlem adı veya fiyat türü adı içinde arama yapar. |
| `@TARIH_BAS`   | NVARCHAR(MAX) | NULL             | Başlangıç tarihi (YYYY-MM-DD formatında olmalıdır). |
| `@TARIH_BIT`   | NVARCHAR(MAX) | NULL             | Bitiş tarihi (YYYY-MM-DD formatında olmalıdır). |
| `@Page`        | INT           | 0                | Sayfalama için kaçıncı sayfanın çekileceği belirtilir. |
| `@ItemCount`   | INT           | 10               | Sayfa başına gösterilecek kayıt sayısı. |

## Çıktı
Stored procedure, aşağıdaki alanları içeren bir sonuç kümesi döndürür:

- **ID**: Muhasebe hareketi kimliği
- **MUSTERI_AD**: Müşteri adı
- **ISLEM_AD**: İşlem türü adı
- **TUTAR**: İşlem tutarı
- **FIYAT_TUR_AD**: Fiyat türü adı
- **FIYAT_TUR_ICON**: Fiyat türü ikonu
- **ACIKLAMA**: Açıklama
- **FATURA**: Fatura numarası
- **TARIH**: İşlem tarihi
- **FIS_ID**: Fiş kimliği
- **ODEME_ID**: Ödeme kimliği
- **TotalCount**: Toplam kayıt sayısı
- **TotalPages**: Toplam sayfa sayısı

## Örnek Çıktı
| ID  | MUSTERI_AD  | ISLEM_AD                      | TUTAR  | FIYAT_TUR_AD | FIYAT_TUR_ICON | ACIKLAMA                                                                                   | FATURA | TARIH                      | FIS_ID   | ODEME_ID | TotalCount | TotalPages |
|-----|-------------|-------------------------------|--------|--------------|----------------|--------------------------------------------------------------------------------------------|--------|----------------------------|----------|----------|------------|------------|
| 238 | Müşteri 1   | Teslimat Fişi Otomatik Kaydı   | 3540.00| Türk Lirası  | ₺              | BULUT SERİSİ 2 adet ürün, 3 m². Ürün birim fiyatı:590,00 Türk Lirası.                      | NULL   | 2025-03-14 12:03:12.167    | 102117  | 0        | 227        | 23         |
| 237 | Müşteri 3   | Teslimat Fişi Otomatik Kaydı   | 2360.00| Türk Lirası  | ₺              | ATLAS SERİSİ 2 adet ürün, 2 m². Ürün birim fiyatı:590,00 Türk Lirası.                       | NULL   | 2025-03-14 12:02:20.927    | 102116  | 0        | 227        | 23         |
| 236 | Müşteri 2   | Teslimat Fişi Otomatik Kaydı   | 2478.00| Türk Lirası  | ₺              | BULUT SERİSİ 1 adet ürün, 4,20 m². Ürün birim fiyatı:590,00 Türk Lirası.                    | NULL   | 2025-03-14 11:13:47.357    | 102115  | 0        | 227        | 23         |
| 238 | Müşteri 1   | Teslimat Fişi Otomatik Kaydı   | 3402.00| Türk Lirası  | ₺              | RENK SERİSİ 1 adet ürün, 5,4 m². Ürün birim fiyatı:630,00 Türk Lirası.                      | NULL   | 2025-03-14 11:13:47.343    | 102115  | 0        | 227        | 23         |
| 236 | Müşteri 2   | Teslimat Fişi Otomatik Kaydı   | 3823.20| Türk Lirası  | ₺              | LOTUS SERİSİ 3 adet ürün, 2,16 m². Ürün birim fiyatı:590,00 Türk Lirası.                    | NULL   | 2025-03-14 11:13:47.343    | 102115  | 0        | 227        | 23         |
| 238 | Müşteri 1   | Teslimat Fişi Otomatik Kaydı   | 3312.00| Türk Lirası  | ₺              | KONFORT SERİSİ 3 adet ürün, 4,80 m². Ürün birim fiyatı:690,00 Türk Lirası.                   | NULL   | 2025-03-14 11:13:47.327    | 102115  | 0        | 227        | 23         |
| 238 | Müşteri 1   | Teslimat Fişi Otomatik Kaydı   | 1652.00| Türk Lirası  | ₺              | ATLAS SERİSİ 1 adet ürün, 2,8 m². Ürün birim fiyatı:590,00 Türk Lirası.                     | NULL   | 2025-03-14 11:13:47.327    | 102115  | 0        | 227        | 23         |
| 236 | Müşteri 2   | Sanal Pos Üzerinden Ödeme   | 6750.00| Türk Lirası  | ₺              | 521807XXXXXXXXXX numaralı kart ile X kullanıcısı tarafından yapılan ödeme.                 | NULL   | 2025-03-14 11:12:10.417    | 0        | 95334    | 227        | 23         |
| 237 | Müşteri 3   | Teslimat Fişi Otomatik Kaydı   | 1953.00| Türk Lirası  | ₺              | PAINT SERİSİ 1 adet ürün, 3,1 m². Ürün birim fiyatı:630,00 Türk Lirası.                     | NULL   | 2025-03-14 11:06:42.657    | 102114  | 0        | 227        | 23         |
| 237 | Müşteri 3   | Teslimat Fişi Otomatik Kaydı   | 3312.00| Türk Lirası  | ₺              | TAÇ SERİSİ 4 adet ürün, 7,20 m². Ürün birim fiyatı:460,00 Türk Lirası.                       | NULL   | 2025-03-14 10:58:45.090    | 102113  | 0        | 227        | 23         |
238 | Müşteri 1 | Teslimat Fişi Otomatik Kaydı | 3312,00	| Türk Lirası	| 	₺ 	| 	TAÇ SERİSİ 4 adet ürün, 7,20 m². Ürün birim fiyatı:460,00 Türk Lirası. | NULL | 2025-03-14 10:58:45.090 | 102113	| 0 | 227 | 23|

## Notlar
- Dinamik SQL kullanılmaktadır.
- `STRING_SPLIT` fonksiyonu kullanılarak birden fazla ID ile sorgu yapılabilir.
- Sayfalama için `OFFSET ... FETCH NEXT` yöntemi uygulanmıştır.
- Tarih dönüşümü için `TRY_CONVERT(DATETIME, @TARIH_BAS, 120)` kullanılmıştır.
