#
# Copyright (C) 2011-2021 Intel Corporation. All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
#
#   * Redistributions of source code must retain the above copyright
#     notice, this list of conditions and the following disclaimer.
#   * Redistributions in binary form must reproduce the above copyright
#     notice, this list of conditions and the following disclaimer in
#     the documentation and/or other materials provided with the
#     distribution.
#   * Neither the name of Intel Corporation nor the names of its
#     contributors may be used to endorse or promote products derived
#     from this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
# "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
# LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
# A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
# OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
# SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
# LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
# DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
# THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
# (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
#
#

TOP_DIR := ../../..
include ../buildenv.mk

AENAME = launch_enclave

# CFLAGS   += -Werror
# CXXFLAGS += -Werror

INCLUDE += -I$(LINUX_PSW_DIR)/ae/common
INCLUDE += -I$(LINUX_PSW_DIR)/ae/data/constants/linux 
INCLUDE += -I$(SGX_HEADER_DIR)/libcxx

SRC := $(wildcard *.cpp)
OBJ := $(SRC:.cpp=.o)
OBJS := $(sort $(OBJ) version.o)
AENAME_OUT :=  $(SONAME) lefuzz le.signed.so #generate $(SONAME) only but do not sign it
SGX_ENCLAVE_SIGNER := $(SGX_SDK)/bin/x64/sgx_sign

CONFIG := config_linux.xml

.PHONY: all
all: $(AENAME_OUT) 

$(SONAME): $(OBJS)
	$(CXX) $(CXXFLAGS) -o $@  $(OBJS) -nostdlib -nodefaultlibs -nostartfiles $(LDTFLAGS)
	$(STRIP) --strip-unneeded --remove-section=.comment --remove-section=.note $(SONAME)

$(OBJ):$(AENAME)_t.c  
$(OBJ): %.o:%.cpp
	$(CXX) $(CXXFLAGS) -fno-exceptions -fno-rtti $(INCLUDE) $(DEFINES) -c $< -o $@

$(AENAME)_t.c: $(EDLFILE)
	@$(EDGER8R) --trusted $< --search-path $(SGX_HEADER_DIR)

le.signed.so: le.so
	@$(SGX_ENCLAVE_SIGNER) sign -key ../ref_le/ref_keys/enclave_private_test_key.pem -enclave $(AENAME) -out $@ -config config_linux.xml
	@echo "SIGN =>  $@"

$(BUILD_DIR):
	@$(MKDIR) $@

# Gen fuzzer
App_C_Flags := -fPIC -Wno-attributes $(INCLUDE) $(DEFINES)

App_Link_Flags := -L$(SGX_LIB_DIR) -lpthread
ifneq ($(SGX_MODE), HW)
	App_Link_Flags += -lsgx_urts_sim
else
	App_Link_Flags += -lsgx_urts
endif

ifneq ($(SGX_MODE), HW)
	App_Link_Flags += -lsgx_uae_service_sim
else
	App_Link_Flags += -lsgx_uae_service
endif

App_Link_Flags += \
	-ldl \
	-Wl,-rpath=$(SGX_LIB_DIR) \
	-Wl,-whole-archive -lSGXSanRTApp -Wl,-no-whole-archive \
	-lSGXFuzzerRT \
	-lcrypto \
	-lboost_program_options \
	-rdynamic

launch_enclave_u.c: $(EDGER8R) launch_enclave.edl
	@$(EDGER8R) --untrusted launch_enclave.edl --search-path ../enclave --search-path $(SGX_SDK)/include --dump-parse ./Enclave.edl.json

launch_enclave_u.o: launch_enclave_u.c
	@echo "CC   <=  $<"
	$(CXX) $(App_C_Flags) -c $< -o $@  \
	-flegacy-pass-manager \
	-Xclang -load -Xclang $(SGX_SDK)/lib64/libSGXFuzzerPass.so

lefuzz: launch_enclave_u.o
	@echo  "Gen fuzzer"
	$(CXX) $^ -o $@ $(App_Link_Flags)


.PHONY: clean
clean:
	@$(RM) *.o
	@$(RM) *.so
	@$(RM) *.map
	@$(RM) *_t.* *_u.*
	@$(RM) $(SONAME).*
	@$(RM) lefuzz Enclave.edl.json
	@rm -f InstrumentStatistics.json

.PHONY: rebuild
rebuild: 
	$(MAKE)	clean
	$(MAKE) all
