# This Makefile can be used with GNU Make or BSD Make

CC ?= gcc

CFLAGS= -Os -Wall -Wextra -Wpedantic -Werror -Wmissing-prototypes -Wredundant-decls -march=native -fPIC -I../../../common $(EXTRAFLAGS)

SOLIB=libmldsa44.so
HEADERS=api.h ntt.h packing.h params.h poly.h polyvec.h reduce.h rounding.h sign.h symmetric.h
OBJECTS=ntt.o packing.o poly.o polyvec.o reduce.o rounding.o sign.o symmetric-shake.o

all: $(SOLIB)

%.o: %.c $(HEADERS)
	$(CC) $(CFLAGS) -c -o $@ $<

$(SOLIB): $(OBJECTS)
	$(CC) -shared -o $@ $(OBJECTS)

clean:
	$(RM) $(OBJECTS)
	$(RM) $(SOLIB)
	$(RM) test_mldsa44 benchmark_mldsa44

test: test_mldsa44
	./test_mldsa44

test_mldsa44: test_mldsa44.c $(SOLIB)
	$(CC) $(CFLAGS) -Wno-missing-prototypes -o test_mldsa44 test_mldsa44.c -L. -lmldsa44 ../../../common/fips202.c -Wl,-rpath,.

benchmark_mldsa44: benchmark_mldsa44.c $(SOLIB)
	$(CC) $(CFLAGS) -Wno-missing-prototypes -o benchmark_mldsa44 benchmark_mldsa44.c -L. -lmldsa44 ../../../common/randombytes.c ../../../common/fips202.c -Wl,-rpath,.
