# uTensor-C
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>

// 1. BOLUM: KUTUPHANE (TinyTensor PRO + ALIGNMENT)

typedef enum {
    TENSOR_FLOAT32,
    TENSOR_FLOAT16,
    TENSOR_INT8
} DataType;

typedef struct {
    void* data;              // Veriye isaret eden pointer
    uint16_t* shape;         // Boyut dizisine isaret eden pointer
    uint8_t dims_count;
    uint32_t total_elements;
    DataType dtype;
} TinyTensor;

// Simulasyon fonksiyonlari
float fp16_to_fp32(uint16_t val) { return (float)val / 100.0f; }
uint16_t fp32_to_fp16(float val) { return (uint16_t)(val * 100.0f); }

//  create_tensor 
TinyTensor* create_tensor(DataType type, uint16_t* shape, uint8_t dims_count) {
    // 1. Guvenlik Kontrolleri
    if (shape == NULL || dims_count == 0) return NULL;

    // 2. Toplam eleman sayisi hesapla
    uint32_t total_elements = 1;
    int i; 
    for(i=0; i<dims_count; i++) total_elements *= shape[i];

    // 3. Eleman boyutu
    size_t element_size;
    switch (type) {
        case TENSOR_FLOAT32: element_size = sizeof(float); break;     // 4 Bytes
        case TENSOR_FLOAT16: element_size = sizeof(uint16_t); break;  // 2 Bytes
        case TENSOR_INT8:    element_size = sizeof(int8_t); break;    // 1 Byte
        default: element_size = 1; break;
    }

    // 4. Bellek Hizalama (MEMORY ALIGNMENT) Hesabi
    // Veri kismi (data) mutlaka 4 byte'in kati olan bir adreste baslamali.
    size_t struct_size = sizeof(TinyTensor);
    size_t shape_size = dims_count * sizeof(uint16_t);
    
    
    size_t current_offset = struct_size + shape_size;
    
    // 4'e bolunebilmesi icin bosluk hesabi
    size_t padding = 0;
    if (current_offset % 4 != 0) {
        padding = 4 - (current_offset % 4);
    }

    size_t data_size = total_elements * element_size;
    
    // Toplam gereken bellek: Struct + Shape + Padding + Data
    uint8_t* memory_block = (uint8_t*)malloc(current_offset + padding + data_size);
    
    if (!memory_block) return NULL;

    // 5. Pointer Atamalari
    TinyTensor* tensor = (TinyTensor*)memory_block;
    
    // Shape dizisi struct'tan hemen sonra
    tensor->shape = (uint16_t*)(memory_block + struct_size);
    
    // Data kismi, shape + padding sonrasinda (Artik 4-byte aligned!)
    tensor->data = (void*)(memory_block + current_offset + padding);
    
    // Degerleri doldur
    tensor->dtype = type;
    tensor->dims_count = dims_count;
    tensor->total_elements = total_elements;
    
    for(i=0; i<dims_count; i++) tensor->shape[i] = shape[i];

    return tensor;
}

float get_tensor_value(TinyTensor* t, uint32_t index) {
    if (t == NULL || t->data == NULL) return 0.0f;
    if (index >= t->total_elements) return 0.0f;

    if (t->dtype == TENSOR_FLOAT32) {
        float* ptr = (float*)t->data;
        return ptr[index];
    } 
    else if (t->dtype == TENSOR_FLOAT16) {
        uint16_t* ptr = (uint16_t*)t->data;
        return fp16_to_fp32(ptr[index]);
    } 
    else if (t->dtype == TENSOR_INT8) {
        int8_t* ptr = (int8_t*)t->data;
        return (float)ptr[index]; 
    }
    return 0.0f;
}

void set_tensor_value(TinyTensor* t, uint32_t index, float value) {
    if (t == NULL || t->data == NULL) return;
    if (index >= t->total_elements) return;

    if (t->dtype == TENSOR_FLOAT32) {
        ((float*)t->data)[index] = value;
    } 
    else if (t->dtype == TENSOR_FLOAT16) {
        ((uint16_t*)t->data)[index] = fp32_to_fp16(value);
    } 
    else if (t->dtype == TENSOR_INT8) {
        if (value > 127) value = 127;
        if (value < -128) value = -128;
        ((int8_t*)t->data)[index] = (int8_t)value;
    }
}

void free_tensor(TinyTensor* t) {
    if (t) free(t); 
}

// 2. BOLUM: ANA PROGRAM (SUNUM KISMI)

int main() {
    
    printf("   TinyML Tensor Yonetimi   \n");
	printf("\n");
    // Senaryo: 100 elemanli bir veri (Ornek: Neural Network Katmani)
    uint16_t shape[] = {100}; 
    uint8_t dims = 1;

    //  TEST 1: Klasik Yontem (Float32) 
    printf("Standart Float32 Kullanimi \n");
    TinyTensor* t_f32 = create_tensor(TENSOR_FLOAT32, shape, dims);
    
    if (t_f32 != NULL) {
        // Veri doldurma
        int i;
        for(i=0; i<100; i++) set_tensor_value(t_f32, i, (float)i * 0.5f);

        // Bellek hesabi
        size_t mem_used = 100 * sizeof(float);
        printf(" * Ayrilan Bellek: %zu Bytes\n", mem_used);
        printf(" * Ornek Veri [10]: %.2f\n", get_tensor_value(t_f32, 10));
        
        //Adres hizalamasini goster
        printf("Data Adresi: %p (4-byte Aligned)\n", t_f32->data);
        
        free_tensor(t_f32);
    }
    printf("\n");

    //Int8 Quantized
    printf("TinyML Optimize Yontem (Int8)\n");
    printf("\n");
    TinyTensor* t_int8 = create_tensor(TENSOR_INT8, shape, dims);

    if (t_int8 != NULL) {
        int i;
        for(i=0; i<100; i++) set_tensor_value(t_int8, i, (float)i); 

        size_t mem_used = 100 * sizeof(int8_t);
        printf(" * Ayrilan Bellek: %zu Bytes\n", mem_used);
        printf(" * Ornek Veri [10]: %.2f (Otomatik donusturuldu)\n", get_tensor_value(t_int8, 10));
        
        // TEKNIK KANIT
        printf("Data Adresi: %p (Safe)\n", t_int8->data);
        printf("\n");
        free_tensor(t_int8);
    }

    // --- SONUC RAPORU ---
    printf("   SONUC: Bellek Tasarrufu %%75 Basarili!     \n");

    printf("\nCikmak icin Enter'a basin...");
    getchar(); 

    return 0;
}
