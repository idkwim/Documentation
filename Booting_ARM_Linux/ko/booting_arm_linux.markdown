#Booting ARM Linux

Vincent Sanders <vince@arm.linux.org.uk>

한글번역: 박영기(리누즈박) <ghostyak@gmail.com>

Review and advice, large chunks of the ARM Linux kernel, all around good guy: **Russell King**

Review, advice and numerous clarifications.: **Nicolas Pitre**

Review and advice: **Erik Mouw, Zwane Mwaikambo, Jeff Sutherland, Ralph Siemsen, Daniel Silverstone, Martin Michlmayr, Michael Stevens, Lesley Mitchell, Matthew Richardson**

Review and referenced information (see bibliography): **Wookey**



Copyright © 2004 Vincent Sanders

- This document is released under a GPL licence.
- All trademarks are acknowledged.

While every precaution has been taken in the preparation of this article, the pub- lisher assumes no responsibility for errors or omissions, or for damages resulting from the use of the information contained herein.

|  VERSION      | DATE                            |     |
| ------------- |:-------------------------------:| --- |
|               | 2004-06-04                      |     |
| Revision 1.00 | Revision History <br/> 10th May 2004  | VRS |
| Revision 1.10 | Initial Release <br/> 4th June 2004   | VRS |


Update example code to be more complete.

Improve wording in places, changes suggested by Nicolas Pitre. Update Section 2, “Other bootloaders”.

Update acknowledgements.

#Table of Contents
1. [About This Document](#About This Document)
2. [Other bootloaders](#Other bootloaders)
3. [Overview](#Overview)
4. [Configuring the system's memory](#Configuring the system's memory)
5. [Loading the kernel image](#Loading the kernel image)
6. [Loading an initial RAM disk](#Loading an initial RAM disk)
7. [Initialising a console](#Initialising a console)
8. [Kernel parameters](#Kernel parameters)
9. [Obtaining the ARM Linux machine type](#Obtaining the ARM Linux machine type)
10. [Starting the kernel](#Starting the kernel)
11. [Tag Reference](#Tag Reference)
 * [ATAG_CORE](#ATAG_CORE)
 * [ATAG_NONE](#ATAG_NONE)
 * [ATAG_MEM](#ATAG_MEM)
 * [ATAG_VIDEOTEXT](#ATAG_VIDEOTEXT)
 * [ATAG_RAMDISK](#ATAG_RAMDISK)
 * [ATAG_INITRD2](#ATAG_INITRD2)
 * [ATAG_SERIAL](#ATAG_SERIAL)
 * [ATAG_REVISION](#ATAG_REVISION)
 * [ATAG_VIDEOLFB](#ATAG_VIDEOLFB)
 * [ATAG_CMDLINE](#ATAG_CMDLINE)
12. [Complete example](#Complete example)
13. [Bibliography](#Bibliography)





<a name="About This Document"></a>
# 1. About This Document

이 문서는 2.4.18버전과 그 이후 버전의 새로운 부팅 절차에 대하여 설명한다. 예전 방식은 더이상 사용되지 않는다.

이 문서는 많은 사람들이 작성한 다양한 소스들을 포함하기 때문에, Maintainer와 ARM Linux 메일링 리스트에 질문하기 전에 이 소스들을 참고하는 것이 좋다. 예전부터 다양한 주제들에 대한 이야기가 반복적으로 이야기 되었다. 기본적인 것들을 숙지하지 않은 상태에서 이야기하는 질문이라면, 그 질문은 무시될 수 있다.

또한 이 문서에서는 ARM Linux를 부팅하기 위해서, 부트로더 구현자가 커널 도입부의 모든 어셈블러를 알 필요가 없다고 가정한다. 많은 부팅 문제들이 있었지만 커널 시작부분의 어셈블러에서는 문제가 발생한 경우는 없었다. 또한 이 코드는 사파의 기술이 잔뜩 들어가 있어서 문제를 해결하는 통찰력을 제공해 주지는 못한다.
















<a name="Other bootloaders"></a>
# 2. Other bootloaders

새 부트로더를 작성하려고 생각중이라면, 이미 존재하는 부트로더 중에서 적절한 녀석이 있는지 고려해 볼 필요가 있다. GPL 로더에서부터 젠장맞을 상용 로더까지 있다. 아래 간략한 목록을 첨부하지만, [Bibliography](#Bibliography)에서 더 많은 솔루션을 얻을 수 있을 것이다.

**Table 1. Bootloaders**

| NAME      | URL                             | DESCRIPTION    |
| --------- |---------------------------------| -------------- |
| Blob      | Blob bootloader <br/> [http://www.sf.net/projects/blob/](http://www.sf.net/projects/blob/) | GPL bootloader for SA11x0 (StrongARM) platforms.    |
| Bootldr   | Bootldr <br/> [http://www.handhelds.org/sources.html](http://www.handhelds.org/sources.html)            | Both GPL and non-GPL versions available, mainly used for handheld devices. |
| Redboot   | Redboot <br/> [http://sources.redhat.com/redboot/](http://sources.redhat.com/redboot/)         | Redhat loader released under their eCos licence. |
| U-Boot    | U-Boot <br/> [http://sourceforge.net/projects/u-boot/](http://sourceforge.net/projects/u-boot/)   | GPL universal bootloader, provides support for several CPUs. |
| ABLE      | ABLE bootloader <br/> [http://www.simtec.co.uk/products/SWABLE/](http://www.simtec.co.uk/products/SWABLE/)   | Commercial bootloader with comprehensive feature set |
















<a name="overview"></a>
# 3. Overview

ARM Linux는 시스템을 초기화 하려면 타겟 머신에 특화된 코드가 필요하다. 이를 위하여 ARM Linux는 아주 적은 코드를 필요로 하지만, 실제로 많은 부트로더들은 별도의 추가적인 기능을 제공하고 있다.

ARM Linux의 최소 요구사항은 다음과 같다:

* 메모리를 설정할 것
* 적절한 위치에 Kernel image를 적재할 것
* 적절한 위치에 Initial RAM disk를 로드할 것 (선택사항)
* 커널에게 넘겨줄 부트 파라메터를 초기화 할 것
* ARM Linux 머신 타입을 얻어올 것
* 적절한 레지스터 값을 갖고 커널로 진입할 것

추가적으로 부트로더는 커널을 위해서 Serial이나 Video 콘솔을 초기화 하기도 한다. 특히 Serial port는 거의 모든 시스템에서 필수적으로 초기화 한다고 봐야한다. 각 단계들은 다음 섹션에서 설명하겠다.



<a name="Configuring the system's memory"></a>
# 4. Configuring the system's memory

부트로더는 커널이 사용할 휘발성 데어터 저장소인 RAM을 초기화 해야 한다. 이 과정은 머신 의존적인 방식으로 수행한다.아마도 자동으로 RAM의 위치와 사이즈를 자동으로 조정하는 내부 알고리즘을 사용하거나, 머신에서 알고있는 RAM의 지식을 사용하거나, 그것도 아니면 부트로더 디자이너가 적절한 방법을 선택해서 사용한다.

이 모든 방법들이 부트로더에 의해 수행되고 있음을 주목해야 한다. 커널은 부트로더가 제공해 주는 것 외엔 RAM 설정에 대해 아무것도 몰라야 한다. 커널 내부에 있는 machine_fixup() 함수는 RAM 설정을 하기위한 가장 확실한 위치가 아니다. 커널과 부트로더의 책임은 분명한 차이가 있다.

물리주소 레이아웃은 ATAG_MEM 파라메터를 사용해서 커널로 전달된다. 메모리는 드문드문 달릴 수 있다. 물론 조각이 최소가 되는 것이 더 좋긴 하지만, 필연적으로 완벽하게 연속적일 필요는 없다. 따라서 여러 메모리의 영역을 사용하기 위하여 ATAG_MEM 블록이 여러개 오는 것이 허용된다. 만약 전달된 영역들이 연속적이면 커널은 전달된 블록들을 하나로 합친다.

또한 부트로더는 커널 커맨드 라인의 `mem=` 파라메터를 사용해서 메모리를 다룰 수 있다. 커맨드라인 옵션에 대해서는 `linux/Documentation/kernel-parameters.txt`에 자세히 설명되어 있다.

커널 커맨드라인 명령어 `mem=`은 `mem=<size>[KM][,@<phys_offset>]`처럼 사용한다. 메모리 영역을 잡기 위하여 사이즈와, 물리 메모리 위치를 잡는 것이 가능하다. 이 명령은 `mem=`파라메터를 여러개 설정함으로써, 다른 offset에 여러 연속된 메모리 블록을 지정하는 것을 허용한다.



<a name="Loading the kernel image"></a>
# 5. Loading the kernel image

커널을 빌드하면 압출되지 않은 "Image" 또는 압축된 "zImage"가 생성된다. 

압축되지 않은 Image 파일은 일반적으로 잘 사용되지 않는다(식별 가능한 Magin Number를 바이너리에 포함하지 않기 때문에). 압축된 zImage 포멧이 일반적인 환경에서 보편적으로 사용된다. 

zImage는 Magic Number이외에도 여러 잇점을 갖고 있다. 보통 외부 저장장치에서 읽는 속도보다 압축해제에 걸리는 시간이 빠르다. 그리고 바이너리에 문제가 생겼을 때 압축해제시 에러가 나기 때문에 Image의 무결성을 보장받을 수 있다. 커널은 내부 구조와 상태를 알고있기 때문에 외부 프로그램에 의한 압축해제 시간보다 더 빨리 압축을 해제시킬 수 있다.

zImage 바이너리의 앞부분에는 Magic Number와 몇 가지 유용한 정보가 있다.

| Offset into zImage | Value           | Description |
| ------------------ |:---------------:|:-----------:|
| 0x24               | 0x016F2818      | ARM Linux zImage를 식별하기 위한 Magic Number |
| 0x28               | 시작 주소         | zImage가 시작하는 주소   |
| 0x2C               | 끝 주소          | zImage가 끝나는 주소     |

시작과 끝 주소는 압축된 Image의 길이를 결정하는데 사용될 수 있다(size = end - start). 만약 커널 Image에 어떤 데이터가 붙어있다면 각종 부트로더 들은 이 정보를 이용해서 Image의 길이를 얻어낸다. 이 데이터는 보통 Initial RAM Disk (initrd)를 위해 사용된다. zImage 코드가 위치 독립적이기 때문에 `시작 주소`는 보통 `0`이다.

zImage 코드는 위치독립(Position Independent Code/PIC)적이기 때문에 적재가능한 위치 어디든지 올라갈 수 있다. 압출해제된 최대 커널 사이즈는 4MB이다. 이 사이즈는 고정되어 있으며 만약 **bootpImage 타겟**<a name="tag5-1"></a> [[1](#tag51-detail)]이 사용된다면, initrd를 포함하는 크기이다.


> <u><b>NOTE</b></u>
> 
> 비록 zImage가 어디든 적재될 수 있지만 주의를 기울여야 한다. zImage는 압축 해제된 Image가 들어갈 추가적인 메모리를 필요로 한다.
> 
> zImage 압축해제 코드는 압축된 데이터를 덮어쓰지 않음을 확인해야 한다. 만약 커널이 이러한 충돌을 발견했다면 압축된 zImage 다음 부분에 즉시 압축을 해제한다. 그리고나서 커널을 재배치한다. zImage가 적재되는 메모리 영역은 그 뒤로 최대 4MB의 여유공간을 확보해야 한다. <a name="tag5-2"></a> [[2](#tag52-detail)]
> 
> 예를들면, 4MB bank 영역 내에 zImage와 ZRELADDR을 같이 두는 것은 아마도 원하는 대로 동작하지 않을 것이다.

사실 zImage가 메모리의 어디든 적재될 수 있는 능력이 있지만, 관습적으로 물리 RAM주소의 `베이스주소 + 0x8000(32KB)` 위치에 적재한다. 이 여분의 공간 내에 offset `0x100(256B)` 위치에 파라메터 블록이 오고, zero page위치에 exception vector가 오고, `0x4000(16KB)` 위치에 Page table이 온다. <a name="tag53"></a> [[3](#tag53-detail)]

이렇게 배치하는 방식은 매우 일반적이다.

----

<a name="tag51-detail"></a>
[[1](#tag51)] BootpImage 타겟: 커널의 빌드 타겟 이름. BootpImage 타겟은 zImage를 만든 후 그 뒤에 initrd의 바이너리를 이어붙인다. 이 때 zImage의 사이즈는 offset 0x28(start), 0x2C(end)을 사용하여 구한다.

<a name="tag52-detail"></a>
[[2](#tag52)] 압축되지 않은 Image의 크기: 

<a name="tag53-detail"></a>
[[3](#tag53)] zero page: 물리 RAM주소의 offset `0x0`부터 '0x100(256B)`까지의 영역을 말한다. 

----

역자 주) 물리 RAM의 스냅샷을 그려보면 다음과 같을 것이다.

````
+------------------+
| zImage           |
+------------------+ 0x0000_8000 (32 KByte)
| Page Table Entry |
+------------------+ 0x0000_4000 (16 KByte)
| Parameters       |
+------------------+ 0x0000_0100 (256 Byte)
| exception vector |
+------------------+ 0x0000_0000
==================== ======================
      구성            물리RAM의 오프셋 주소
````






<a name="Loading an initial RAM disk"></a>
# 6. Loading an initial RAM disk

initial RAM disk는 많은 시스템에서 사용된다. 별도의 드라이버나 설정 없이 root filesystem을 사용할 수 있기 때문이다. 자세한 정보는 [linux/Documentation/initrd.txt](http://lxr.free-electrons.com/source/Documentation/initrd.txt) 에서 얻을 수 있다.

ARM Linux에서 initial RAM disk를 사용하는 방법은 두 가지이다. 첫째, bootpImage 타겟을 지정해서 커널을 빌드하면 빌드타임에 zImage 파일 뒤쪽에 initrd가 이어붙어서 이미지가 만들어진다. 이 방법은 initrd를 사용하기 위하여 부트로더의 도움이 필요 없다는 장점이 있으나, RAM disk를 메모리에 적재하기 위하여 커널 빌드 프로세스가 물리 주소에 대한 지식을 갖추고 있어야 한다는 단점이 있다(INITRD_PHYS 정의를 사용해서). 압축되지 않은 커널과 initrd의 크기를 합쳐서 4MB의 제한이 있다. 이 제한 때문에 bootpImage는 실제로 거의 사용되지 않는다. 

두 번째, 다른 미디어로부터 initrd를 읽어서 RAM에 적재시키는 역할을 부트로더가 담당하는 방법이다. 이 방법이 훨씬 많이 사용된다. 이 위치는 ATAG_INITRD2와 ATAG_RAMDISK를 통해서 커널로 전달된다.

전통적으로 initrd는 물리 메모리의 시작주소로 부터 8MB 떨어진 곳에 위치한다. 어디에 적재되든 간에 initrd를 압축해제하기 위해서 충분한 메모리가 있어야 한다. 예를들면, 최소`zImage + 압축해제된 zImage + initrd + 압축해제된 ramdisk` 만큼의 공간이 필요하다는 말이다. initrd의 압축이 해제되면 압축된 initrd가 사용했던 메모리는 반환된다. ramdisk의 위치가 가지는 제한은 다음과 같다:

- 반드시 단일 메모리 영역에 위치할 것 (ATAG_MEM 파라메터에 의해 나뉜 영역에 배치하지 말라는 의미)
- page 단위로 align되어 있을 것 (보통 4KB)
- 커널의 압축을 해제하는 zImage 코드와 충돌을 피할 것 (이 부분은 확인하지 않으므로 덮어씌워질 수 있음)








<a name="Initialising a console"></a>
# 7. Initialising a console


콘솔은 커널이 시스템을 초기화 할 때 무엇을 수행하고 있는지 보기 위한 방법으로 강력히 추천되는 방법이다. 여러가지 적합한 입출력가능한 드라이버들이 있겠지만, 가장 일반적인 방법은 비디오 프레임 버퍼를 사용하거나 시리얼 드라이버를 사용하는 것이다. ARM Linux가 실행되는 시스템은 거의 대부분 시리얼 콘솔 포트를 제공하는 경향이 있다.

부트로더는 타겟 보드에서 시리얼 포트를 초기화 하고 활성화 시켜야만 한다. 물론 하드웨어 전원관리같은 것들을 포함해서 말이다.
커널 시리얼 드라이버는 커널 콘솔이 어떤 포트를 사용해야 하는지 자동으로 감지할 수 있다. (일반적으로 디버깅/통신 목적으로 사용)

또다른 방식은 부트로더가 `consol=` 파라메터로 포트넘버와 시리얼 포멧을 커널에 전달하는 것이다. 자세한 사항은 [linux/Documentation/kernel-parameters.txt](http://lxr.free-electrons.com/source/Documentation/kernel-parameters.txt)에 설명되어 있다.





<a name="Kernel parameters"></a>
# 8. Kernel parameters






<a name="Obtaining the ARM Linux machine type"></a>
# 9. Obtaining the ARM Linux machine type

부트로더가 제공해야하는 추가적인 정보는 `머신타입`이다. 머신타입은 종종 MACH_TYPE 으로 정의되는 간단한 숫자이다.

머신타입 넘버는 ARM Linux 웹사이트의 [Machine Registry](http://www.arm.linux.org.uk/developer/machines/)에서 얻을 수 있다. 머신타입은 가능하면 프로젝트 라이프 사이클에서 아주 일찍 획득해야만 한다. 커널을 포팅(머신 정의 같은)하는데에 여러가지 영향이 있기 때문이다. 그리고 만약 나중에 머신타입을 바꾼다면 원치않은 많은 이슈들이 생겨날 것이다. 이 값들은 커널소스([linux/arch/arm/tools/mach-type](http://lxr.free-electrons.com/source/arch/arm/tools/mach-types))에 정리되어 있다.

부트로더는 머신타입을 하드코딩하던지 하드웨어를 통해서 읽던지 간에, 어떠한 방법을 써서라도 반드시 머신타입의 값을 획득해야만 한다. 구현은 시스템에 따라 완전히 다를 수 있기 때문에 이 문서에서는 더이상 언급하지 않겠다.



<a name="Starting the kernel"></a>
# 10. Starting the kernel

부트로더가 커널을 시작하기 전에 해야하는 작업들을 마쳤다면, CPU register에 올바른 값들을 저장한 후 커널을 실행시켜야 합니다.

요구사항은 다음과 같습니다:

- CPU는 반드시 IRQ/FIQ 인터럽트 중지 상태의 SVC(supervisor) 모드로 진입할 것
- MMU를 반드시 off시켜야 함 (주소 변환 없이 물리 RAM에서 실행되어야 함)
- Data 캐시를 반드시 off시켜야 함
- Instruction 캐시는 on 또는 off되어도 상관 없음
- CPU register 0은 반드시 `0`이어야 함
- CPU register 1은 반드시 `ARM Linux 머신타입` 이어야 함
- CPU register 2는 반드시 파라메터 목록을 가리키는 물리 주소여야 함

이제 부트로더는 커널 이미지의 첫 번째 명령으로 점프하는 작업을 하므로써, 커널 이미지를 호출한다.








<a name="Tag Reference"></a>
# 11. Tag Reference






<a name="ATAG_CORE"></a>
## ATAG_CORE
ATAG_CORE -- Start tag used to begin list

**Value**
0x54410001

**Size**
5 (2 if no data)

**Structure members**

````
struct atag_core {
u32 flags; /* bit 0 = read-only */
u32 pagesize; /* systems page size (usually 4k) */
u32 rootdev; /* root device number */
};
````

**Description**

This tag must be used to start the list, it contains the basic information any bootloader must pass, atag length of 2 indicates the tag has no structure attached.
















<a name="ATAG_NONE"></a>
## ATAG_NONE
ATAG_NONE -- Empty tag used to end list

**Value**

0x00000000

**Size**

2

**Structure members**

None

**Description**

This tag is used to indicate the list end. It is unique in that its size field in the header should be set to 0 (not 2).
















<a name="ATAG_MEM"></a>
## ATAG_MEM

ATAG_MEM -- Tag used to describe a physical area of memory.

**Value**

0x54410002

**Size**

4

**Structure members**

````
struct atag_mem {
u32 size; /* size of the area */
u32 start; /* physical start address */
};
````

**Description**

Describes an area of physical memory the kernel is to use.
















<a name="ATAG_VIDEOTEXT"></a>
## ATAG_VIDEOTEXT

ATAG_VIDEOTEXT -- Tag used to describe VGA text type displays

**Value**

0x54410003

**Size**

5

**Structure members**

````
struct atag_videotext {
u8 x; /* width of display */
u8 y; /* height of display */
u16 video_page;
u8 video_mode;
u8 video_cols;
u16 video_ega_bx;
u8 video_lines;
u8 video_isvga;
u16 video_points;
};
````

**Description**
















<a name="ATAG_RAMDISK"></a>
## ATAG_RAMDISK

ATAG_RAMDISK -- Tag describing how the ramdisk will be used by the kernel

**Value**

0x54410004

**Size**

5

**Structure members**

````
struct atag_ramdisk {
u32 flags; /* bit 0 = load, bit 1 = prompt */
u32 size; /* decompressed ramdisk size in _kilo_ bytes */
u32 start; /* starting block of floppy-based RAM disk image */
};
````

**Description**

Describes how the (initial) ramdisk will be configured by the kernel, specifically this allows for the bootloader to ensure the ramdisk will be large enough to take the decompressed initial ramdisk image the bootloader is passing using ATAG_INITRD2.














<a name="ATAG_INITRD2"></a>
## ATAG_INITRD2

ATAG_INITRD2 -- Tag describing the physical location of the compressed ramdisk image

**Value**

0x54420005

**Size**

4

**Structure members**

````
struct atag_initrd2 {
u32 start; /* physical start address */
u32 size; /* size of compressed ramdisk image in bytes */
};
````

**Description**

Location of a compressed ramdisk image, usually combined with an ATAG_RAMDISK. Can be used as an initial root file system with the addition of a command line parameter of 'root=/dev/ram'. This tag supersedes the original ATAG_INITRD which used virtual addressing, this was a mistake and produced issues on some systems. All new bootloaders should use this tag in preference.












<a name="ATAG_SERIAL"></a>
## ATAG_SERIAL

ATAG_SERIAL -- Tag with 64 bit serial number of the board

**Value**

0x54410006

**Size**

4

**Structure members**

````
struct atag_serialnr {
u32 low;
u32 high;
};
````

**Description**












<a name="ATAG_REVISION"></a>
## ATAG_REVISION

ATAG_REVISION -- Tag for the board revision

**Value**

0x54410007

**Size**

3

**Structure members**

````
struct atag_revision {
u32 rev;
};
````

**Description**












<a name="ATAG_VIDEOLFB"></a>
## ATAG_VIDEOLFB

ATAG_VIDEOLFB -- Tag describing parameters for a framebuffer type display

**Value**

0x54410008

**Size**

8

**Structure members**

````
struct atag_videolfb {
u16 lfb_width;
u16 lfb_height;
u16 lfb_depth;
u16 lfb_linelength;
u32 lfb_base;
u32 lfb_size;
u8 red_size;
u8 red_pos;
u8 green_size;
u8 green_pos;
u8 blue_size;
u8 blue_pos;
u8 rsvd_size;
u8 rsvd_pos;
};
````

**Description**












<a name="ATAG_CMDLINE"></a>
## ATAG_CMDLINE

ATAG_CMDLINE -- Tag used to pass the commandline to the kernel

**Value**

0x54410009

**Size**

2 + ((length_of_cmdline + 3) / 4)

**Structure members**

````
struct atag_cmdline {
char cmdline[1]; /* this is the minimum size */
};
````

**Description**

Used to pass command line parameters to the kernel. The command line must be NULL terminated. The length_of_cmdline variable should include the terminator.














<a name="Complete example"></a>
# 12. Complete example


다음 예제는 이 문서에서 설명하고 있는 모든 정보들을 포함하는 심플한 부트로더의 소스이다. 실제 부트로더는 좀 더 많은 코드를 요구하지만, 이 예제를 통해 무엇이 필요지 참고하기 바란다. 이 코드는 BSD licence 아래에서 배포된다. 필요하다면 무료로 복사하고 사용할 수 있다.

````
/* example.c
 * example ARM Linux bootloader code
 * this example is distributed under the BSD licence
 */
/* list of possible tags */
#define ATAG_NONE 0x00000000
#define ATAG_CORE 0x54410001
#define ATAG_MEM 0x54410002
#define ATAG_VIDEOTEXT 0x54410003
#define ATAG_RAMDISK 0x54410004
#define ATAG_INITRD2 0x54420005
#define ATAG_SERIAL 0x54410006
#define ATAG_REVISION 0x54410007
#define ATAG_VIDEOLFB 0x54410008
#define ATAG_CMDLINE 0x54410009
/* structures for each atag */
struct atag_header {
	u32 size; /* length of tag in words including this header */
	u32 tag; /* tag type */
};
struct atag_core {
	u32 flags;
	u32 pagesize;
	u32 rootdev;
};
struct atag_mem {
	u32 size;
	u32 start;
};
struct atag_videotext {
	u8 x;
	u8 y;
	u16 video_page;
	u8 video_mode;
	u8 video_cols;
	u16 video_ega_bx;
	u8 video_lines;
	u8 video_isvga;
	u16 video_points;
};
struct atag_ramdisk {
	u32 flags;
	u32 size;
	u32 start;
};
struct atag_initrd2 {
	u32 start;
	u32 size;
};
struct atag_serialnr {
	u32 low;
	u32 high;
};
struct atag_revision {
	u32 rev;
};
struct atag_videolfb {
	u16 lfb_width;
	u16 lfb_height;
	u16 lfb_depth;
	u16 lfb_linelength;
	u32 lfb_base;
	u32 lfb_size;
	u8 red_size;
	u8 red_pos;
	u8 green_size;
	u8 green_pos;
	u8 blue_size;
	u8 blue_pos;
	u8 rsvd_size;
	u8 rsvd_pos;
};
struct atag_cmdline {
	char cmdline[1];
};
struct atag {
	struct atag_header hdr;
	union {
		struct atag_core core;
		struct atag_mem mem;
		struct atag_videotext videotext;
		struct atag_ramdisk ramdisk;
		struct atag_initrd2 initrd2;
		struct atag_serialnr serialnr;
		struct atag_revision revision;
		struct atag_videolfb videolfb;
		struct atag_cmdline cmdline;
	} u;
};
#define tag_next(t) ((struct tag *)((u32 *)(t) + (t)->hdr.size))
#define tag_size(type) ((sizeof(struct tag_header) + sizeof(struct type)) >> 2)
static struct atag *params; /* used to point at the current tag */

static void
setup_core_tag(void * address,long pagesize)
{
	params = (struct tag *)address; /* Initialise parameters to start at params->hdr.tag = ATAG_CORE; /* start with the core tag */
	params->hdr.size = tag_size(atag_core); /* size the tag */
	params->u.core.flags = 1; /* ensure read-only */
	params->u.core.pagesize = pagesize; /* systems pagesize (4k) */
	params->u.core.rootdev = 0; /* zero root device (typicaly overidden params = tag_next(params); /* move pointer to next tag */
}

static void
setup_ramdisk_tag(u32_t size)
{
	params->hdr.tag = ATAG_RAMDISK; /* Ramdisk tag */
	params->hdr.size = tag_size(atag_ramdisk); /* size tag */
	params->u.ramdisk.flags = 0; /* Load the ramdisk */
	params->u.ramdisk.size = size; /* Decompressed ramdisk size */
	params->u.ramdisk.start = 0; /* Unused */
	params = tag_next(params); /* move pointer to next tag */
}

static void
setup_initrd2_tag(u32_t start, u32_t size)
{
	params->hdr.tag = ATAG_INITRD2; /* Initrd2 tag */
	params->hdr.size = tag_size(atag_initrd2); /* size tag */
	params->u.initrd2.start = start; /* physical start */
	params->u.initrd2.size = size; /* compressed ramdisk size */
	params = tag_next(params); /* move pointer to next tag */
}

static void
setup_mem_tag(u32_t start, u32_t len)
{
	params->hdr.tag = ATAG_MEM; /* Memory tag */
	params->hdr.size = tag_size(atag_mem); /* size tag */
	params->u.mem.start = start; /* Start of memory area (physical address) params->u.mem.size = len; /* Length of area */
	params = tag_next(params); /* move pointer to next tag */
}

static void
setup_cmdline_tag(const char * line)
{
	int linelen = strlen(line);
	if(!linelen)
		return; /* do not insert a tag for an empty params->hdr.tag = ATAG_CMDLINE; /* Commandline tag */
	params->hdr.size = (sizeof(struct atag_header) + linelen + 1 + 4) >> 2;
	strcpy(params->u.cmdline.cmdline,line); /* place commandline into tag */
	params = tag_next(params); /* move pointer to next tag */
}

static void
setup_end_tag(void)
{
	params->hdr.tag = ATAG_NONE; /* Empty tag ends list */
	params->hdr.size = 0; /* zero length */
}
#define DRAM_BASE 0x10000000
#define ZIMAGE_LOAD_ADDRESS DRAM_BASE + 0x8000
#define INITRD_LOAD_ADDRESS DRAM_BASE + 0x800000

static void
setup_tags(parameters)
{
	setup_core_tag(parameters, 4096); /* standard core tag 4k pagesize */
	setup_mem_tag(DRAM_BASE, 0x4000000); /* 64Mb at 0x10000000 */
	setup_mem_tag(DRAM_BASE + 0x8000000, 0x4000000); /* 64Mb at 0x18000000 */
	setup_ramdisk_tag(4096); /* create 4Mb ramdisk */
	setup_initrd2_tag(INITRD_LOAD_ADDRESS, 0x100000); /* 1Mb of compressed data setup_cmdline_tag("root=/dev/ram0"); /* commandline setting root device */
	setup_end_tag(void); /* end of tags */
}

int
start_linux(char *name,char *rdname)
{
	void (*theKernel)(int zero, int arch, u32 params);
	u32 exec_at = (u32)-1;
	u32 parm_at = (u32)-1;
	u32 machine_type;
	exec_at = ZIMAGE_LOAD_ADDRESS;
	parm_at = DRAM_BASE + 0x100
		load_image(name, exec_at); /* copy image into RAM */
	load_image(rdname, INITRD_LOAD_ADDRESS);/* copy initial ramdisk image into RAM setup_tags(parm_at); /* sets up parameters */
	machine_type = get_mach_type(); /* get machine type */
	irq_shutdown(); /* stop irq */
	cpu_op(CPUOP_MMUCHANGE, NULL); /* turn MMU off */
	theKernel = (void (*)(int, int, u32))exec_at; /* set the kernel address */
	theKernel(0, machine_type, parm_at); /* jump to kernel with register set return 0;
}
````






<a name="Bibliography"></a>
# 13. Bibliography

ARM Linux website Documentation [[http://www.arm.linux.org.uk/developer/](http://www.arm.linux.org.uk/developer/)]. Russell M King.

Linux Kernel Documentation/arm/booting.txt [[http://www.arm.linux.org.uk/developer/booting.php](http://www.arm.linux.org.uk/developer/booting.php)]. Russell M King.

Setting R2 correctly for booting the kernel
[[http://lists.arm.linux.org.uk/pipermail/linux-arm-kernel/2003-January/013126.html](http://lists.arm.linux.org.uk/pipermail/linux-arm-kernel/2003-January/013126.html)]
(explanation of booting requirements). Russell M King.

Wookey's post summarising booting
[[http://lists.arm.linux.org.uk/pipermail/linux-arm-kernel/2002-April/008700.html](http://lists.arm.linux.org.uk/pipermail/linux-arm-kernel/2002-April/008700.html)].
Wookey.

Makefile defines and symbols
[[http://lists.arm.linux.org.uk/pipermail/linux-arm-kernel/2001-July/004064.html](http://lists.arm.linux.org.uk/pipermail/linux-arm-kernel/2001-July/004064.html)]. Russell
M King.

Bootloader guide [[http://www.aleph1.co.uk/armlinux/docs/ARMbooting/t1.html](http://www.aleph1.co.uk/armlinux/docs/ARMbooting/t1.html)]. Wookey.

Kernel boot order
[[http://lists.arm.linux.org.uk/pipermail/linux-arm-kernel/2001-October/005212.html](http://lists.arm.linux.org.uk/pipermail/linux-arm-kernel/2001-October/005212.html)]. Russell
M King.

Advice for head.S Debugging
[[http://lists.arm.linux.org.uk/pipermail/linux-arm-kernel/2002-January/006730.html](http://lists.arm.linux.org.uk/pipermail/linux-arm-kernel/2002-January/006730.html)]. Russell
M King.

Linux kernel 2.4 startup [[http://billgatliff.com/articles/emb-linux/startup.html/index.html](http://billgatliff.com/articles/emb-linux/startup.html/index.html)]. Bill Gatliff.

Blob bootloader [[http://www.sf.net/projects/blob/](http://www.sf.net/projects/blob/)]. Erik Mouw.

Blob bootloader on lart [[http://www.lart.tudelft.nl/lartware/blob/](http://www.lart.tudelft.nl/lartware/blob/)]. Erik Mouw.
