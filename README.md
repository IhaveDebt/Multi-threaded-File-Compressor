#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <string.h>

#define CHUNK_SIZE 1024

typedef struct {
    char *data;
    int start;
    int end;
} ThreadData;

void *compress_chunk(void *arg) {
    ThreadData *td = (ThreadData *)arg;
    char *src = td->data + td->start;
    int len = td->end - td->start;
    char *out = malloc(len * 2);
    int j = 0;

    for (int i = 0; i < len; i++) {
        int count = 1;
        while (i + 1 < len && src[i] == src[i + 1]) { count++; i++; }
        j += sprintf(out + j, "%c%d", src[i], count);
    }

    fwrite(out, 1, j, stdout);
    free(out);
    pthread_exit(NULL);
}

int main(int argc, char *argv[]) {
    if (argc < 2) { printf("Usage: %s <file>\n", argv[0]); return 1; }

    FILE *fp = fopen(argv[1], "r");
    if (!fp) { perror("Error opening file"); return 1; }

    fseek(fp, 0, SEEK_END);
    long size = ftell(fp);
    rewind(fp);

    char *buffer = malloc(size);
    fread(buffer, 1, size, fp);
    fclose(fp);

    int nthreads = 4;
    pthread_t threads[nthreads];
    ThreadData td[nthreads];
    int chunk = size / nthreads;

    for (int i = 0; i < nthreads; i++) {
        td[i].data = buffer;
        td[i].start = i * chunk;
        td[i].end = (i == nthreads - 1) ? size : (i + 1) * chunk;
        pthread_create(&threads[i], NULL, compress_chunk, &td[i]);
    }

    for (int i = 0; i < nthreads; i++) pthread_join(threads[i], NULL);

    free(buffer);
    return 0;
}
