TinyTensor: Gömülü Sistemler için Optimize Edilmiş Dinamik Tensör KütüphanesiTinyTensor, RAM kısıtlı mikrodenetleyiciler (Arduino, ESP32, STM32 vb.) üzerinde TinyML (Gömülü Yapay Zeka) modellerini çalıştırmak için tasarlanmış; bellek güvenliği (memory safety) ve performans odaklı saf C kütüphanesidir.Bu proje, standart float dizilerinin yarattığı bellek şişkinliğini önlemek için Quantization (Nicemleme) ve Dinamik Bellek Hizalama (Memory Alignment) tekniklerini kullanır.⚡ Temel Özellikler💾 Tek Blok Bellek Tahsisi (Single Malloc Architecture): Struct, Shape bilgisi ve Data yığınını tek bir malloc çağrısı ile ardışık (contiguous) bellekte tutar. Heap parçalanmasını (fragmentation) önler.📏 ARM Uyumlu Bellek Hizalama (4-Byte Alignment): Veri bloğunun başlangıç adresini otomatik hesaplanan Padding ile 4-byte sınırına hizalar. Bu, ARM işlemcilerde Hard Fault hatalarını engeller.📉 Quantization Desteği: 32-bit Float veriler yerine 8-bit Integer kullanarak %75'e varan bellek tasarrufu sağlar.🔄 Dinamik Tip Dönüşümü (Type Erasure): void* işaretçiler ve çalışma zamanı (runtime) tip kontrolü ile çok biçimli (polymorphic) veri yönetimi sunar.🛠️ Bellek Mimarisi (Memory Layout)Bu kütüphane, bellekte dağınık duran işaretçiler yerine, optimize edilmiş tek bir blok kullanır:[ TinyTensor Struct ] + [ Shape Array (uint16_t) ] + [ Padding (0-3 Bytes) ] + [ DATA BLOCK (Aligned) ]
^                                                                            ^
Base Pointer                                                          Data Pointer (4-Byte Aligned)
Bu yapı sayesinde işlemci önbelleği (Cache Locality) en verimli şekilde kullanılır.🚀 Kurulum ve DerlemeBu proje standart C99 kütüphaneleri ile yazılmıştır. Herhangi bir ek bağımlılık gerektirmez.Windows (MinGW / GCC)Proje dizininde terminali açın ve şu komutu çalıştırın:Bashgcc -std=c99 main.c -o tensor_app
./tensor_app
Linux / macOSBashgcc main.c -o tensor_app
./tensor_app
💻 Örnek KullanımC#include "tinytensor.h" // (Kodun entegre oldugu varsayilmistir)

int main() {
    // 100 Elemanlı Bir Tensör Tanımla
    uint16_t shape[] = {100};
    uint8_t dims = 1;

    // --- Durum 1: Yüksek Hassasiyet (Float32) ---
    TinyTensor* t_float = create_tensor(TENSOR_FLOAT32, shape, dims);
    set_tensor_value(t_float, 0, 3.14f);
    
    // --- Durum 2: Bellek Tasarrufu (Int8 Quantized) ---
    TinyTensor* t_quant = create_tensor(TENSOR_INT8, shape, dims);
    set_tensor_value(t_quant, 0, 3.14f); // Otomatik sıkıştırma

    // Okuma yaparken otomatik tip dönüşümü (De-quantization)
    float val = get_tensor_value(t_quant, 0); 
    
    // Temizlik (Tek seferde tüm blok temizlenir)
    free_tensor(t_float);
    free_tensor(t_quant);
}
📊 Performans KarşılaştırmasıAşağıdaki sonuçlar, 100 elemanlı bir katman için yapılan testlerden alınmıştır:Veri TipiBellek Kullanımı (Veri)DurumFloat32 (Standart)400 Bytes❌ Yüksek TüketimInt8 (TinyTensor)100 Bytes✅ %75 TasarrufNot: TinyTensor, sadece veri boyutunu küçültmekle kalmaz, aynı zamanda Struct ve Metadata için gereken ek yükü de tek blokta optimize eder.👨‍💻 Geliştirici NotlarıBu proje, gömülü sistemlerde Pointer Arithmetic (İşaretçi Aritmetiği) ve Memory Safety (Bellek Güvenliği) konularını derinlemesine uygulamak amacıyla geliştirilmiştir. Özellikle create_tensor fonksiyonundaki padding algoritması, donanım seviyesinde optimizasyon sağlar.
