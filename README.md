#include <stdio.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>


typedef struct {
    char     ChunkID[4];      
    uint32_t ChunkSize;      
    char     Format[4];       
    char     Subchunk1ID[4];  
    uint32_t Subchunk1Size;    
    uint16_t AudioFormat;      
    uint16_t NumChannels;
    uint32_t SampleRate;
    uint32_t ByteRate;
    uint16_t BlockAlign;
    uint16_t BitsPerSample;
    char     Subchunk2ID[4];   
    uint32_t Subchunk2Size;   
} WavHeader;



#define WINDOW_SIZE 101

void filter(float *input, uint32_t duration)
{
    if (duration == 0) return;

    float *temp_output = (float *)malloc(duration * sizeof(float));

    if (temp_output == NULL) {
        return;
    }

    int i, j;
    float sum;
    const int half_window = WINDOW_SIZE / 2;

    
    for (i = 0; i < half_window && i < duration; i++) {
        temp_output[i] = input[i];
    }

    for (i = half_window; i < duration - half_window; i++) {
        sum = 0.0f;
        for (j = -half_window; j <= half_window; j++) {
            sum += input[i + j];
        }
        temp_output[i] = sum / (float)WINDOW_SIZE;
    }

  
    for (i = duration - half_window; i < duration; i++) {
        temp_output[i] = input[i];
    }

    for (i = 0; i < duration; i++) {
        input[i] = temp_output[i];
    }

    free(temp_output);
}



int main() {
    const char *input_filename = "input.wav";
    const char *output_filename = "output.wav";
    FILE *infile, *outfile;
    WavHeader header;
    int16_t* audio_data_int16 = NULL;
    uint32_t data_chunk_size = 0;
    uint32_t num_channels, num_frames, bytes_per_sample;

    infile = fopen(input_filename, "rb");
    if (infile == NULL) {
        fprintf(stderr, "Error: Cannot open input file %s\n", input_filename);
        return 1;
    }

   
    fread(header.ChunkID, 1, 4, infile);
    fread(&header.ChunkSize, 4, 1, infile);
    fread(header.Format, 1, 4, infile);
    if (strncmp(header.ChunkID, "RIFF", 4) != 0 || strncmp(header.Format, "WAVE", 4) != 0) {
        fprintf(stderr, "Error: Not a valid WAVE file.\n");
        fclose(infile);
        return 1;
    }

   
    char chunk_id[4];
    uint32_t chunk_size;
    int found_fmt = 0;
    int found_data = 0;

    while (fread(chunk_id, 1, 4, infile) == 4) {
        fread(&chunk_size, 4, 1, infile);

        if (strncmp(chunk_id, "fmt ", 4) == 0) {
            // Found the format chunk, read its contents
            header.Subchunk1Size = chunk_size;
            fread(&header.AudioFormat, 2, 1, infile);
            fread(&header.NumChannels, 2, 1, infile);
            fread(&header.SampleRate, 4, 1, infile);
            fread(&header.ByteRate, 4, 1, infile);
            fread(&header.BlockAlign, 2, 1, infile);
            fread(&header.BitsPerSample, 2, 1, infile);
            // Skip any extra format bytes
            if (header.Subchunk1Size > 16) {
                fseek(infile, header.Subchunk1Size - 16, SEEK_CUR);
            }
            found_fmt = 1;
        } else if (strncmp(chunk_id, "data", 4) == 0) {
            // Found the data chunk, read the audio data
            data_chunk_size = chunk_size;
            audio_data_int16 = (int16_t*)malloc(data_chunk_size);
            if (audio_data_int16 == NULL) {
                fprintf(stderr, "Error: Memory allocation failed for audio data.\n");
                fclose(infile);
                return 1;
            }
            fread(audio_data_int16, 1, data_chunk_size, infile);
            found_data = 1;
        } else {
          
            fseek(infile, chunk_size, SEEK_CUR);
        }

        if (found_fmt && found_data) {
            break; 
        }
    }
    fclose(infile);

    if (!found_data || !found_fmt) {
        fprintf(stderr, "Error: Could not find format and data chunks in WAV file.\n");
        if (audio_data_int16) free(audio_data_int16);
        return 1;
    }

   
    if (header.AudioFormat != 1) {
        fprintf(stderr, "Error: Only PCM (uncompressed) format is supported.\n");
        free(audio_data_int16);
        return 1;
    }
    if (header.BitsPerSample != 16) {
        fprintf(stderr, "Error: Only 16-bit audio is supported.\n");
        free(audio_data_int16);
        return 1;
    }
    num_channels = header.NumChannels;
    bytes_per_sample = header.BitsPerSample / 8;  
    num_frames = data_chunk_size / (num_channels * bytes_per_sample);
    if (num_frames * num_channels * bytes_per_sample != data_chunk_size) {
        fprintf(stderr, "Error: Invalid data chunk size (not aligned).\n");
        free(audio_data_int16);
        return 1;
    }

    float **audio_channels = (float **)malloc(num_channels * sizeof(float *));
    if (audio_channels == NULL) {
        fprintf(stderr, "Error: Memory allocation failed for channel buffers.\n");
        free(audio_data_int16);
        return 1;
    }
    for (uint32_t ch = 0; ch < num_channels; ch++) {
        audio_channels[ch] = (float *)malloc(num_frames * sizeof(float));
        if (audio_channels[ch] == NULL) {
            fprintf(stderr, "Error: Memory allocation failed for channel %u.\n", ch);
            // Free previous
            for (uint32_t c = 0; c < ch; c++) free(audio_channels[c]);
            free(audio_channels);
            free(audio_data_int16);
            return 1;
        }
    }


    for (uint32_t f = 0; f < num_frames; f++) {
        for (uint32_t ch = 0; ch < num_channels; ch++) {
            int16_t sample = audio_data_int16[f * num_channels + ch];
            audio_channels[ch][f] = (float)sample / 32768.0f;
        }
    }

    printf("Applying improved filter (window size %d) to %u channel(s), %u frames...\n",
           WINDOW_SIZE, num_channels, num_frames);
    for (uint32_t ch = 0; ch < num_channels; ch++) {
        filter(audio_channels[ch], num_frames);
    }
    printf("Filtering complete.\n");

   
    for (uint32_t f = 0; f < num_frames; f++) {
        for (uint32_t ch = 0; ch < num_channels; ch++) {
            float sample_float = audio_channels[ch][f];
           
            if (sample_float > 1.0f) sample_float = 1.0f;
            if (sample_float < -1.0f) sample_float = -1.0f;
            audio_data_int16[f * num_channels + ch] = (int16_t)(sample_float * 32767.0f);
        }
    }

  
    for (uint32_t ch = 0; ch < num_channels; ch++) {
        free(audio_channels[ch]);
    }
    free(audio_channels);

   
    outfile = fopen(output_filename, "wb");
    if (outfile == NULL) {
        fprintf(stderr, "Error: Cannot create output file %s\n", output_filename);
        free(audio_data_int16);
        return 1;
    }
    
    
    strncpy(header.ChunkID, "RIFF", 4);
    strncpy(header.Format, "WAVE", 4);
    strncpy(header.Subchunk1ID, "fmt ", 4);
    strncpy(header.Subchunk2ID, "data", 4);
    header.Subchunk1Size = 16;
    header.AudioFormat = 1;  // PCM
    header.NumChannels = (uint16_t)num_channels;
    header.SampleRate = header.SampleRate;  // Preserve original
    header.BitsPerSample = 16;
    header.BlockAlign = (uint16_t)(num_channels * bytes_per_sample);
    header.ByteRate = header.SampleRate * header.BlockAlign;
    header.Subchunk2Size = data_chunk_size;
    header.ChunkSize = 36 + data_chunk_size;  // Standard header size + data

    fwrite(&header, sizeof(WavHeader), 1, outfile);
    fwrite(audio_data_int16, data_chunk_size, 1, outfile);
    fclose(outfile);

   
    free(audio_data_int16);

    printf("Success! Improved filtered audio saved to %s\n", output_filename);
    printf("Note: The larger filter window (101) provides better noise reduction for low-amplitude noise.\n");
    printf("If you need even more aggressive filtering or a different type (e.g., median for spikes), provide more details!\n");
    return 0;
}
