## Post #1
- Username: Savage
- Rank: VIP member
- Number of posts: 559
- Joined date: Sun Apr 17, 2005 6:00 pm
- Post datetime: 2006-08-17T20:40:37+00:00
- Post Title: Electronic Arts Audio format (SChl, exa, etc...)

Maybe you know the post of 
[Harry Potter and the Goblet of Fire](http://forum.xentax.com/viewtopic.php?t=1932)
There i talk about the audio format of EaGames, and how to convert to wav and viceversa using a special tool

With this doc (not mine ) maybe the people can understand better how works this audio codec (my the first )


```
	    Electronic Arts New Audio File Formats Description	    5-01-2002
=============================================================================

By Valery V. Anisimovsky (samael@avn.mccme.ru)

In this document I'll try to describe audio file formats used in many (new)
Electronic Arts games. Described are formats for music, movie soundtracks and
partly sound effects/speech.

The games using these formats include: Need For Speed 2, NFS3, NFS4, NFS5,
NBA Live'98, NBA'99, NBA'2000, NHL Online'98, NHL'99, NHL'2000, NHL'2001,
FIFA'98, FIFA'99, FIFA'2000, FIFA'2001, Bundesliga Stars 2000, Madden NFL'98,
Madden NFL'99, Madden NFL'2000, EURO'2000, World Cup 98, Triple Play 99,
Fighter Pilot, World War II Fighters, Warhammer II: Dark Omen, Dungeon Keeper 2,
Populous 3, Wing Commander: Prophecy. Maybe many more, e.g.: NBA'97, FIFA'97.

The files this document deals with have extensions: .ASF, .STR, .MUS, .LIN,
.MAP, .WVE, .TGQ, .DCT, .MAD, .UV, .UV2, .BNK, .VIV. Note that the files
described here may have other extensions (and the same structure!):
Electronic Arts tends to change extensions from game to game.

Throughout this document I use C-like notation.

All numbers in all structures described in this document are stored in files
using little-endian (Intel) byte order, unless otherwise stated.

=========================
1. .ASF/.STR Music Files
=========================

The music in many new Electronic Arts games is in .ASF stand-alone files
(sometimes ASF files have extension .STR). These files have the block
structure analoguous to RIFF. Namely, these files are divided into blocks
(without any global file header like RIFFs have). Each block has the
following header:

struct ASFBlockHeader
{
  char	szBlockID[4];
  DWORD dwSize;
};

szBlockID -- string ID for the block.

dwSize -- size of the block (in bytes) INCLUDING this header.

Further I'll describe the contents of blocks of all block types in .ASF file.

When I say "block begins with..." that means "the contents of that
block (which begin just after ASFBlockHeader) begin with...".
Quoted strings are block IDs.

"SCHl": header block. This is the first block in ASF.
In the most of files this block begins with the ID string "PT\0\0" (or number
0x50540000). Further goes the PT header data which describes audio data in
the file. This PT header should be parsed rather than just read as a simple
structure. Here I give the parsing code. These functions use fread() and fseek()
stdio functions.

// first of all, we need a function which reads a small (variable) number
// bytes and composes a DWORD of them. Note that such DWORD will be a kind
// of big-endian (Motorola) stored, e.g. 3 consecutive bytes 0x12 0x34 0x56
// will give a DWORD 0x00123456.
DWORD ReadBytes(FILE* file, BYTE count)
{
  BYTE	i, byte;
  DWORD result;

  result=0L;
  for (i=0;i<count;i++)
  {
    fread(&byte,sizeof(BYTE),1,file);
    result<<=8;
    result+=byte;
  }

  return result;
}

// these will be set by ParsePTHeader
DWORD dwSampleRate;
DWORD dwChannels;
DWORD dwCompression;
DWORD dwNumSamples;
DWORD dwDataStart;
DWORD dwLoopOffset;
DWORD dwLoopLength;
DWORD dwBytesPerSample;
BYTE  bSplit;
BYTE  bSplitCompression;

// Here goes the parser itself
// This function assumes that the current file pointer is set to the
// start of PT header data, that is, just after PT string ID "PT\0\0"
void ParsePTHeader(FILE* file)
{
  BYTE byte;
  BOOL bInHeader, bInSubHeader;

  bInHeader=TRUE;
  while (bInHeader)
  {
    fread(&byte,sizeof(BYTE),1,file);
    switch (byte) // parse header code
    {
      case 0xFF: // end of header
	   bInHeader=FALSE;
      case 0xFE: // skip
      case 0xFC: // skip
	   break;
      case 0xFD: // subheader starts...
	   bInSubHeader=TRUE;
	   while (bInSubHeader)
	   {
	     fread(&byte,sizeof(BYTE),1,file);
	     switch (byte) // parse subheader code
	     {
	       case 0x82:
		    fread(&byte,sizeof(BYTE),1,file);
		    dwChannels=ReadBytes(file,byte);
		    break;
	       case 0x83:
		    fread(&byte,sizeof(BYTE),1,file);
		    dwCompression=ReadBytes(file,byte);
		    break;
	       case 0x84:
		    fread(&byte,sizeof(BYTE),1,file);
		    dwSampleRate=ReadBytes(file,byte);
		    break;
	       case 0x85:
		    fread(&byte,sizeof(BYTE),1,file);
		    dwNumSamples=ReadBytes(file,byte);
		    break;
	       case 0x86:
		    fread(&byte,sizeof(BYTE),1,file);
		    dwLoopOffset=ReadBytes(file,byte);
		    break;
	       case 0x87:
		    fread(&byte,sizeof(BYTE),1,file);
		    dwLoopLength=ReadBytes(file,byte);
		    break;
	       case 0x88:
		    fread(&byte,sizeof(BYTE),1,file);
		    dwDataStart=ReadBytes(file,byte);
		    break;
	       case 0x92:
		    fread(&byte,sizeof(BYTE),1,file);
		    dwBytesPerSample=ReadBytes(file,byte);
		    break;
	       case 0x80: // ???
		    fread(&byte,sizeof(BYTE),1,file);
		    bSplit=ReadBytes(file,byte);
		    break;
	       case 0xA0: // ???
		    fread(&byte,sizeof(BYTE),1,file);
		    bSplitCompression=ReadBytes(file,byte);
		    break;
	       case 0xFF:
		    subflag=FALSE;
		    flag=FALSE;
		    break;
	       case 0x8A: // end of subheader
		    bInSubHeader=FALSE;
	       default: // ???
		    fread(&byte,sizeof(BYTE),1,file);
		    fseek(file,byte,SEEK_CUR);
	     }
	   }
	   break;
      default:
	   fread(&byte,sizeof(BYTE),1,file);
	   if (byte==0xFF)
	      fseek(file,4,SEEK_CUR);
	   fseek(file,byte,SEEK_CUR);
    }
  }
}

dwSampleRate -- sample rate for the file. Note that headers of most of
ASFs/MUSes I've seen DO NOT contain sample rate subheader section. Currently
I just set sample rate for such files to the default: 22050 Hz. It seems to
work okay.

dwChannels -- number of channels for the file: 1 for mono, 2 for stereo.
If this is NOT set by ParsePTHeader, then you may use the default: stereo.

dwCompression -- Compression tag. If this is 0x00, then no compression is
used and audio data is signed 16-bit PCM. If this is 0x07, the audio data is
compressed with EA ADPCM algorithm. Please read the next section for the
description of EA ADPCM decompression scheme. In some files this tag is
omitted -- I use 0x00 (no compression) for them.

dwNumSamples -- number of samples in the file.

dwDataStart -- in ASF files this's not used.

dwLoopOffset -- offset when looping (from start of sound part).

dwLoopLength -- length when looping.

dwBytesPerSample -- bytes per sample (Default is 2). Divide this by
dwChannels to get resolution of sound data.

bSplit -- this looks like to be 0x01 for files using "split" SCDl blocks
(see below). If this subheader field is absent, the file uses "normal"
(interleaved) SCDl blocks.

bSplitCompression -- this looks like to be 0x08 for files using non-compressed
"split" SCDl blocks. If this subheader field is absent in the file using
"split" SCDls, the file uses EA ADPCM compression. This subheader field
should not appear in a file using "normal" (interleaved) SCDls.

The structure and the meanings of some parts of PT header is very uncertain.
Please mail me if you find out more!

Note that some music/video files have somewhat different format of SCHl header.
Namely, first comes PATl header: it begins with "PATl" ID string and its size
is 56 bytes (always?) including its ID string. After PATl header comes TMpl
header:

struct TMplHeader
{
  char	szID[4];
  BYTE	bUnknown1;
  BYTE	bBits;
  BYTE	bChannels;
  BYTE	bCompression;
  WORD	wUnknown2;
  WORD	wSampleRate;
  DWORD dwNumSamples; // ???
  BYTE	bUnknown3[20];
};

szID -- string ID, always "TMpl".

bBits -- resolution of sound data (0x10 for 16-bit, 0x8 for 8-bit).

bChannels -- channels number: 1 for mono, 2 for stereo.

bCompression -- if 0x00, the data in the file is not compressed: signed 8-bit
PCM or signed 16-bit PCM. If this byte is 0x02, the audio data is compressed
with IMA ADPCM. See my EA-ASF.TXT specs for description of IMA ADPCM
decompression scheme.

wSampleRate -- sample rate for the file.

dwNumSamples -- number of samples in the file. May be used for song length
(in seconds) calculation. Should be divided by 2 for mono sound. Note that
the meaning of this field may be different when TMpl header is used inside
the SCHl header.

"SCCl": count block. This block goes after "SCHl" and contains one DWORD
value which is a number of "SCDl" data blocks in ASF file.

"SCDl": data block. These blocks contain audio data. Depending on the
parameters set in the header (see above) SCDl block may contain compressed
(by EA ADPCM or IMA ADPCM) or non-compressed audio data and the data itself
may be interleaved or split (see below).

If no compression is used and the file does not use "split" SCDl blocks,
SCDl block begins with a DWORD value which is the number of samples in this
block and after that comes signed 16-bit PCM data, in the interleaved form:
LRLR...LR (L and R are 16-bit sample values for left and right channels).

Hereafter by "chunk" I mean the audio data in the "SCDl" data block, that is,
compressed/non-compressed data which starts after chunk header.

In the newer EA games (NHL'2000/NBA'2000/FIFA'99'2000/NFS5) non-compressed
"split" SCDl blocks are used. These blocks begin with a chunk header:

struct ASFSplitPCMChunkHeader
{
  DWORD dwOutSize;
  DWORD dwLeftChannelOffset;
  DWORD dwRightChannelOffset;
}

dwOutSize -- size of audio data in this chunk (in samples).

dwLeftChannelOffset, dwRightChannelOffset -- offsets to PCM data for
left and right channels, relative to the byte which immediately follows
ASFSplitPCMChunkHeader structure. E.g. for left channel this offset is zero
-- the data starts immediately after this structure.

After this structure comes PCM data for stereo wavestream and it's not
interleaved (LRLRLR...), but it's split: first go sample values for left
channel, then -- for right channel, that is the layout is LL...LRR...R.

If EA ADPCM (or IMA ADPCM) compression is used, but the file does not use
"split" SCDls, each SCDl block begins with a chunk header:

struct ASFChunkHeader
{
  DWORD dwOutSize;
  SHORT lCurSampleLeft;
  SHORT lPrevSampleLeft;
  SHORT lCurSampleRight;
  SHORT lPrevSampleRight;
};

dwOutSize -- size of decompressed audio data in this chunk (in samples).

lCurSampleLeft, lCurSampleRight, lPrevSampleLeft, lPrevSampleRight are initial
values for EA ADPCM decompression routine for this data block (for left and right
channels respectively). I'll describe the usage of these further when I get to
EA ADPCM decompression scheme.

Note that the structure above is ONLY for stereo files. For mono there're
just no lCurSampleRight, lPrevSampleRight fields.

If IMA ADPCM compression is used, the meanings of some chunk header fields
are different -- see my EA-ASF.TXT specs for details.

After this chunk header the compressed data comes. See the next section for
EA ADPCM decompression scheme description.

If EA ADPCM (or IMA ADPCM) compression is used and the file uses "split" SCDls,
each SCDl block begins with a different chunk header:

struct ASFSplitChunkHeader
{
  DWORD dwOutSize;
  DWORD dwLeftChannelOffset;
  DWORD dwRightChannelOffset;
};
SHORT lCurSampleLeft;
SHORT lPrevSampleLeft;
BYTE  bLeftChannelData[]; // compressed data for left channel goes here...
SHORT lCurSampleRight;
SHORT lPrevSampleRight;
BYTE  bRightChannelData[]; // compressed data for right channel goes here...

dwOutSize -- size of decompressed audio data in this chunk (in samples).

dwLeftChannelOffset, dwRightChannelOffset -- offsets to compressed data for
left and right channels, relative to the byte which immediately follows
ASFSplitChunkHeader structure. E.g. for left channel this offset is zero --
the data starts immediately after this structure.

lCurSampleLeft, lCurSampleRight, lPrevSampleLeft, lPrevSampleRight have the
same meaning as above, but note that these values are SHORTs.

So, use mono decoder for each channel data and then create normal LRLR...
stereo waveform before outputting.
Such (newer) files may be separated from the others by presence of 0x80 type
section in PT header (the value stored in the section is 0x01 for such files).
Some of such files also do not contain compression type (0x83) section in
their PT header.

"SCLl": loop block. This block defines looping point for the song. It
contains only DWORD value, which is the looping jump position (in samples)
relative to the start of the song. You should make the jump just when you
encounter this block.

"SCEl": end block. This block indicates the end of audio stream.

Note that in some games audio files are contained within game resources. As
a rule, such resources are not compressed/encrypted, so you may just search
for ASF file signature (e.g. "SCHl") and this will mark the beginning of audio
stream, while "SCEl" block marks the end of that stream.

====================================
2. EA ADPCM Decompression Algorithm
====================================

During the decompression four LONG variables must be maintained for stereo
stream: lCurSampleLeft, lCurSampleRight, lPrevSampleLeft, lPrevSampleRight
and two -- for mono stream: lCurSample, lPrevSample. At the beginning of each
"SCDl" data block you must initialize these variables using the values in
ASFChunkHeader.
Note that LONG here is signed.

Here's the code which decompresses one "SCDl" block of EA ADPCM compressed
stereo stream.

BYTE  InputBuffer[InputBufferSize]; // buffer containing audio data of "SCDl" block
BYTE  bInput;
DWORD dwOutSize; // outsize value from the ASFChunkHeader
DWORD i, bCount, sCount;
LONG  c1left,c2left,c1right,c2right,left,right;
BYTE  dleft,dright;

DWORD dwSubOutSize=0x1c;

i=0;

// process integral number of (dwSubOutSize) samples
for (bCount=0;bCount<(dwOutSize/dwSubOutSize);bCount++)
{
  bInput=InputBuffer[i++];
  c1left=EATable[HINIBBLE(bInput)];   // predictor coeffs for left channel
  c2left=EATable[HINIBBLE(bInput)+4];
  c1right=EATable[LONIBBLE(bInput)];  // predictor coeffs for right channel
  c2right=EATable[LONIBBLE(bInput)+4];
  bInput=InputBuffer[i++];
  dleft=HINIBBLE(bInput)+8;   // shift value for left channel
  dright=LONIBBLE(bInput)+8;  // shift value for right channel
  for (sCount=0;sCount<dwSubOutSize;sCount++)
  {
    bInput=InputBuffer[i++];
    left=HINIBBLE(bInput);  // HIGHER nibble for left channel
    right=LONIBBLE(bInput); // LOWER nibble for right channel
    left=(left<<0x1c)>>dleft;
    right=(right<<0x1c)>>dright;
    left=(left+lCurSampleLeft*c1left+lPrevSampleLeft*c2left+0x80)>>8;
    right=(right+lCurSampleRight*c1right+lPrevSampleRight*c2right+0x80)>>8;
    left=Clip16BitSample(left);
    right=Clip16BitSample(right);
    lPrevSampleLeft=lCurSampleLeft;
    lCurSampleLeft=left;
    lPrevSampleRight=lCurSampleRight;
    lCurSampleRight=right;

    // Now we've got lCurSampleLeft and lCurSampleRight which form one stereo
    // sample and all is set for the next input byte...
    Output((SHORT)lCurSampleLeft,(SHORT)lCurSampleRight); // send the sample to output
  }
}

// process the rest (if any)
if ((dwOutSize % dwSubOutSize) != 0)
{
  bInput=InputBuffer[i++];
  c1left=EATable[HINIBBLE(bInput)];   // predictor coeffs for left channel
  c2left=EATable[HINIBBLE(bInput)+4];
  c1right=EATable[LONIBBLE(bInput)];  // predictor coeffs for right channel
  c2right=EATable[LONIBBLE(bInput)+4];
  bInput=InputBuffer[i++];
  dleft=HINIBBLE(bInput)+8;   // shift value for left channel
  dright=LONIBBLE(bInput)+8;  // shift value for right channel
  for (sCount=0;sCount<(dwOutSize % dwSubOutSize);sCount++)
  {
    bInput=InputBuffer[i++];
    left=HINIBBLE(bInput);  // HIGHER nibble for left channel
    right=LONIBBLE(bInput); // LOWER nibble for right channel
    left=(left<<0x1c)>>dleft;
    right=(right<<0x1c)>>dright;
    left=(left+lCurSampleLeft*c1left+lPrevSampleLeft*c2left+0x80)>>8;
    right=(right+lCurSampleRight*c1right+lPrevSampleRight*c2right+0x80)>>8;
    left=Clip16BitSample(left);
    right=Clip16BitSample(right);
    lPrevSampleLeft=lCurSampleLeft;
    lCurSampleLeft=left;
    lPrevSampleRight=lCurSampleRight;
    lCurSampleRight=right;

    // Now we've got lCurSampleLeft and lCurSampleRight which form one stereo
    // sample and all is set for the next input byte...
    Output((SHORT)lCurSampleLeft,(SHORT)lCurSampleRight); // send the sample to output
  }
}

HINIBBLE and LONIBBLE are higher and lower 4-bit nibbles:
#define HINIBBLE(byte) ((byte) >> 4)
#define LONIBBLE(byte) ((byte) & 0x0F)
Note that depending on your compiler you may need to use additional nibble
separation in these defines, e.g. (((byte) >> 4) & 0x0F).

EATable is the table given in the next section of this document.

Output() is just a placeholder for any action you would like to perform for
decompressed sample value.

Clip16BitSample is quite evident:

LONG Clip16BitSample(LONG sample)
{
  if (sample>32767)
     return 32767;
  else if (sample<-32768)
     return (-32768);
  else
     return sample;
}

As to mono sound, it's just analoguous: dwSubOutSize=0x0E for mono and
you should get predictor coeffs and shift from one byte:

bInput=InputBuffer[i++];
c1=EATable[HINIBBLE(bInput)];	// predictor coeffs
c2=EATable[HINIBBLE(bInput)+4];
d=LONIBBLE(bInput)+8;  // shift value

And also you should process HIGHER nibble of the input byte first and then
LOWER nibble for mono sound.

Of course, this decompression routine may be greatly optimized.

==================
3. EA ADPCM Table
==================

LONG EATable[]=
{
  0x00000000,
  0x000000F0,
  0x000001CC,
  0x00000188,
  0x00000000,
  0x00000000,
  0xFFFFFF30,
  0xFFFFFF24,
  0x00000000,
  0x00000001,
  0x00000003,
  0x00000004,
  0x00000007,
  0x00000008,
  0x0000000A,
  0x0000000B,
  0x00000000,
  0xFFFFFFFF,
  0xFFFFFFFD,
  0xFFFFFFFC
};

==================================================
4. .WVE/.DCT/.MAD/.TGQ/.UV/.UV2 Movie Soundtracks
==================================================

.WVE/.DCT/.MAD/.TGQ/.UV/.UV2 movies have the block structure analoguous to
that of .ASF. Video-related data is in "pIQT", "mTCD", "MADk", "MADm", "MADe",
"pQGT", etc. blocks and sound-related data is just in the same blocks as in
.ASF: "SCHl", "SCCl", "SCDl", "SCLl", "SCEl".
So, to play .WVE/.DCT/.MAD/.TGQ/.UV/.UV2 movie soundtrack, just walk blocks
chain, skip video blocks and process sound blocks. Note that in some games
video files (as well as audio files) are contained within game resources. As
a rule, such resources are not compressed/encrypted, so you may just search
for ASF file signature (e.g. "SCHl") and this will mark the beginning of audio
stream, while "SCEl" block marks the end of that stream.

===================
5. MUS Music Files
===================

Interactive music is in .MUS files. These have the same block structure as
.ASFs with two important differences:
1) MUS file may contain several "SCHl" header blocks.
2) Each "SCHl" header block starts at the position which is a multiple of 4.
That is, if you've read the "SCEl" end block and your current file position is,
say, dwCurPos, do the following: if ((dwCurPos % 4) == 0) just read the next
block, otherwise skip (4 - (dwCurPos % 4)) bytes and then read the next block.

If you walk the block chain of a .MUS file, you'll get the block sequence
like this:
SCHl, SCCl, SCDl, ..., SCEl, SCHl, SCCl, SCDl, ..., SCEl, ....
That is, a MUS file is a kind of collection of ASF files, each ASF file
beginning being aligned on DWORD boundary. Each ASF file starts with "SCHl"
block and ends with "SCEl" block. Further I'll refer to such ASFs in .MUS as
"MUS sections". Each MUS section contains a part of song. If you try to
play these parts consecutively as they appear in .MUS you will not get right
song playback for most .MUS files. To play .MUS in the right sequence you'll
need either .LIN or .MAP file (with the same name) which should be found in
the same directory as the .MUS on Electronic Arts game CD.

While in NFS 2 almost all .MUSes have the correspondent .ASFs which are used
for non-interactive playback, in NFS 3 all songs are .MUSes and to play them
you'll need to use correspondent .LIN file (for some songs -- .MAP file).

=============================================
6. .LIN/.MAP Files and Correct .MUS Playback
=============================================

.LIN/.MAP files which should be found in the same directory as .MUSes define
the interactive and non-interactive ("normal") playback sequences. Typically,
.LINs define normal (non-interactive) and .MAPs define interactive sequences.
Some .MAPs define normal sequence. Both .LINs and .MAPs have the same
structure, which I'll describe here.

Each .LIN or .MAP corresponds to the .MUS with the same name: e.g. CREDITS.MAP
corresponds to CREDITS.MUS and EMPRROCK.LIN -- to EMPRROCK.MUS.

.LIN/.MAP file has the following header:

struct MAPHeader
{
  char szID[4];
  BYTE bUnknown1;
  BYTE bFirstSection;
  BYTE bNumSections;
  BYTE bRecordSize; // ???
  BYTE Unknown2[3];
  BYTE bNumRecords;
};

szID -- string ID, always "PFDx".

bFirstSection -- index (zero-based) of the first MUS section to be played.
Hereafter by "index of .MUS section" I mean the number which identifies
the section in .MUS file: index 0 corresponds to the first section, 1 -- to
the second, etc. That is, the section index is zero-based.

bNumSections -- number of sections in the correspondent MUS file.

bRecordSize -- size of record, array of which follows the table of section
definitions in .LIN/.MAP file. More about this later.

bNumRecords -- number of records in the array mentioned above.

Following the header, comes the table of (bNumSections) definitions for each
section of .MUS. Each definition describes the correspondent .MUS section:
the first describe first .MUS section, the second describes second .MUS
section, etc. Each definition has the following format:

struct MAPSectionDef
{
  BYTE bIndex;
  BYTE bNumRecords;
  BYTE szID[2];
  struct MAPSectionDefRecord msdRecords[8];
};

bIndex -- ??? not necessary for non-interactive playback.

bNumRecords -- number of MAPSectionDefRecords used (of 8) in msdRecords[].
Used are msdRecords[0], ..., msdRecords[bNumRecords-1], others are zeroed.
For .LINs/.MAPs, defining non-interactive playback sequence, it seems that
(bNumRecords) is always 1, that is, only the first MAPSectionDefRecord is used
and should be used for playback sequence. If (bNumRecords) is zero, this means
that the section described by the MAPSectionDef is the final in playback
sequence and there's no next section for it.

szID -- ID, seems to be always "\xFF\xFF". Not necessary for non-interactive
playback.

msdRecords -- array of 8 records (used are only first (bNumRecords)), each
record having the following format:

struct MAPSectionDefRecord
{
  BYTE bUnknown;
  BYTE bMagic;
  BYTE bNextSection;
};

bMagic -- seems to be 0x64 for the records defining non-interactive playback.
But, maybe, not necessarily. Just ignore that.

bNextSection -- index (zero-based) of the next section in the .MUS playback
sequence. The section with the index (bNextSection) should be played after the
section which is described by this MAPSectionDef. More about the .MUS
playback later.

After the table of .MUS section definitions comes the array of
(MAPHeader.bNumRecords) seemingly useless records each record having the size
(MAPHeader.bRecordSize). I've got some doubts about my treatment of
(MAPHeader.bRecordSize) field, so it seems to be safer to use 0x10 as the record
size. Just skip this array. It's of no use for non-interactive playback.

After that array comes the final part of .LIN/.MAP -- the array of DWORDs
which are just the starting positions of .MUS sections (that is, positions
for "SCHl" blocks describing the correspondent sections). Important note:
these DWORDs are stored using big-endian byte order! That means that the four
bytes in the file, e.g., 0x12 0x34 0x56 0x78 constitute the DWORD value
0x12345678 and NOT 0x78563412 (as it's treated by Intel processors). These
starting positions are relative to the .MUS file beginning.

Now, when we know the structure of .LIN/.MAP files, I'll describe how they
should be used for non-interactive .MUS playback.

First, read the .LIN/.MAP header. This gives you the index of first section
in playback sequence (MAPHeader.bFirstSection).
Then get the starting position of this section from the positions table:
fseek(mapfile,sizeof(MAPHeader)+MAPHeader.bNumSections*sizeof(MAPSectionDef)+
MAPHeader.bNumRecords*MAPHeader.bRecordSize+index*sizeof(DWORD),SEEK_SET);
fread(&dwStart,sizeof(DWORD),1,mapfile);
Invert byte order in dwStart: dwStart=SWAPDWORD(dwStart), where
#define SWAPDWORD(x) ((((x)&0xFF)<<24)+(((x)>>24)&0xFF)+(((x)>>8)&0xFF00)+(((x)<<8)&0xFF0000))
Now you've got correct dwStart and just set the file pointer in .MUS file to
that to get to the section start. Read the section's "SCHl" header and
further blocks and play the section.
Then get to this section's definition structure, for example, using the code
like this:
fseek(mapfile,sizeof(MAPHeader)+index*sizeof(MAPSectionDef),SEEK_SET);
Read the section definition:
fread(&secdef,sizeof(MAPSectionDef),1,mapfile);
Now (secdef.msdRecords[secdef.bNumRecords-1].bNextSection) is the next
section to play back. Get its starting position from the table, etc.
Repeat this procedure until you come across either a section you've already
played or the section definition with zero (bNumRecords).
In the former case you may loop the song or just stop playback.
In the latter case you should just stop playback.

Some final words about .MUS/.ASF/.LIN/.MAP files...
When to play .MUS file using .LIN or .MAP and what to use: .LIN or .MAP ?
If along with the .MUS file there's an .ASF file with same name, play the
.ASF file -- it should be used for non-interactive playback.
If there's no .ASF file with the same name as .MUS, but along with the .MUS
there's a .LIN file with the same name as .MUS, play .MUS file using that
.LIN file.
If there's no .LIN or .ASF file correspondent to .MUS file, but there's a .MAP
file with the same name, play the .MUS file using that .MAP.
And finally, if there's none of .ASF, .LIN or .MAP file for .MUS, it's an
error. You may try to play that .MUS section-by-section or use playback
sequence of your choice.

====================================
7. Sound Effects in .BNK/.VIV Files
====================================

Most of sound effects and speech files (and sometimes ASF music files) are
stored in .BNK and .VIV resource files. The .BNK file may contain several
sounds. BNKs of older version have the following header:

struct OldBNKHeader
{
  char	szID[4];
  WORD	wVersion;
  WORD	wNumberOfSounds;
  DWORD dwFirstSoundStart;
  DWORD dwSoundsArray[wNumberOfSounds];
};

For the newer BNK files the header is:

struct NewBNKHeader
{
  char	szID[4];
  WORD	wVersion;
  WORD	wNumberOfSounds;
  DWORD dwFirstSoundStart;
  DWORD dwSoundSize; // = total filesize - dwFirstSoundStart
  DWORD dwUnknown;   // seems to contain small number <20 or -1
  DWORD dwSoundsArray[wNumberOfSounds];
};

szID -- string ID, always "BNKl".

wVersion -- for old version this is 0x0002, for new version -- 0x0004.

wNumberOfSounds -- number of sounds stored in .BNK file.

dwFirstSoundStart -- the starting position of the first sound audio data
relative the BNK file beginning. There's no real use of this...

dwSoundsArray -- the array of (wNumberOfSounds) DWORDs. Each of these is the
shift to the PT header describing the separate sound in .BNK relative to the
starting position of this DWORD. That is, if such DWORD (dwShift) starts at
the position (dwShiftPos) (relative to the start of .BNK), the correspondent
PT header starts at the position: dwPTHeaderPos=dwShiftPos+dwShift.
Note that some DWORDs in this array are zeroes that means they correspond to
no sound. Remember that PT header starts with the "PT\0\0" signature.

So, (dwSoundsArray) points to a number of PT headers in .BNK, which follow the
BNK header. Each of these PT headers describe a separate sound in .BNK.
Refer to the .ASF file description for details on dealing with PT headers.
Note that some PT headers do not contain (dwChannels), (dwSampleRate),
(dwCompression) data. I use the default value if it's omitted in the header:
mono, 22050 Hz, unknown compression. In any case, PT header for .BNK sound
should contain values for (dwNumSamples) and (dwDataStart). (dwDataStart) is
the starting position of sound data relative to the start of .BNK file. Sound
data itself has no additional headers and in case of EA ADPCM compression
(dwCompression==0x07) should be decoded just like "SCDl" block data (following
ASFChunkHeader). As to the size of the sound data, just use (dwNumSamples) and
stop playback of the sound when it's exhausted.

As to .VIV files these seem to be multi-data resources. In particular, they
can contain .BNK/.ASF files. So, if you want to play sounds from a .BNK file
contained within .VIV, just search .VIV for "BNKl" string ID and that will
be just the .BNK file described above. Note that all (dwDataShifts) given in
PT headers in .BNK are always positions relative to the start of .BNK file,
that is, if .BNK is in .VIV, they will be relative to the start of "BNKl"
signature you found in .VIV. To play .ASF file from .VIV you may just search
for "SCHl" string ID and that'll mark the beginning of .ASF file, while
the end will be marked by "SCEl" block.

===========
8. Credits
===========

Dmitry Kirnocenskij (ejt@mail.ru)
Worked out EA ADPCM decompression algorithm.

Toni Wilen (nhlinfo@nhl-online.com)
http://www.nhl-online.com/nhlinfo/
Provided me with info on new SCDl structure, new BNK version header, PATl
and TMpl headers. Toni Wilen is the author of SNDVIEW utility (available
on his pages) which decompresses Electronic Arts audio files and compresses
WAVs back into EA formats.

Jesper Juul-Mortensen (jjm@danbbs.dk, ICQ#43452941)
http://www.danbbs.dk/~jjm
http://nfstoolbox.homepage.dk
http://nfscheats.com/nfstoolbox
Additional info on PT header block types. The author of utilities for NFS'x.

-------------------------------------------

Valery V. Anisimovsky (samael@avn.mccme.ru)
http://bim.km.ru/gap/
http://www.anxsoft.newmail.ru
http://anx.da.ru
On these sites you can find my GAP program which can search for audio files
in .BNK/.VIV resources, and play back .ASF/.MUS/.STR songs, some .BNK/.VIV
sounds and soundtracks of .WVE/.DCT/.MAD/.TGQ/.UV/.UV2 movies.
There's also complete source code of GAP and all its plug-ins there,
including MUS/ASF plug-in, which could be used for further details on how
you can deal with these formats.

```


I attach the program called GAP (Game Audio Player v1.32) it's 700kb's,  if somebody knows a newer version tellme 

[http://www.filefactory.com/file/0dab29/](http://www.filefactory.com/file/0dab29/)
