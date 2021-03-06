# Individual phase work

### This code implements the following instructions:
#### - LDM{Suffix}{Condition} Rn(!), {reglist}
#### - STM{Suffix}{Condition} Rn(!), {reglist}
#### LDM stands for: Load Multiple Registers (from memory)
#### STM stands for: Store Multiple Registers (to memory)





The code is broken down into _4 major parts_:
- _Tokenizer_ (module TokenizeOperands)
- _Parser_ (module Memory)
- _Execution_ (module Execution)
- _Testing_ (module TestFile)


### Tokenizer: 
_Input_ : 
```F#
str : string
```
_Output_ : 
```F#
Token list
```

###### Type created: 
```F#
type Token = |Reg of RName*int 
             |WriteB of int 
             |Comma of int 
             |LCBRA of int 
             |RCBRA of int 
             |Hyphen of int 
             |END
```

Supposed to tokenise Operands only (Opcode detection is already given).


###### Where: 
- ```WriteB``` is to detect ```!``` , which means we write back the final memory address into the base register, Rn.
- ```LCBRA``` and ```RCBRA``` stand for ```Left Curly Bracket``` and ```Right Curly Bracket```.
- ```Hyphen``` is to detect ``` - ``` , since the list of registers can be of the form ``` {R4, R5-LR} ```.
It converts all input operands to uppercase letters to make sure registers such as ```lr``` or ```pc``` will be correctly tokenised.


### Parser:
###### Types created:
```F#
type Root = |LDM
            |STM
	    
type Suffix = |DA
              |DB
              |IA
              |IB
              |ED
              |FA
              |FD
              |EA  
     
type Instr = {instrClass:InstrClass; root: Root; suffix: Suffix; Rn: RName; writeB: bool; reglist: RName list}	    
```


_Inputs_ : 
```F#
ls : LineData
```
_Output_ : 
```F#
Result<Parse<Instr>,string> option
```

The parser is built based around this _simple reasoning_: the syntax for an LDM/STM instruction will ALWAYS have the form of ```OpCode Rn(!), {list of registers}```. Therefore, the list of possible instructions is very straightforward and easy to match. 

Parser breaks the input line into 2 major blocks, which are analysed in the ```parse``` and ```makeRegList``` functions.

The role of the ```parse``` function is to detect a correct syntax starting with an ```Rn(!), {``` block. If it doesn't recognise a correct line, returns an error. Otherwise, it calls ```makeRegList``` on 
```list of registers}```, which is the rest of the input line. This function also checks for syntax error and returns _the list of RName type correctly ordered and with only one appearance of each RName_ (in case reglist is of the form  ```{R1,R1,R1}```). 
All this is done with ```removeDuplicates``` , ```orderListreg``` and ```expandRegRange``` (which from ```R0-R3``` creates ```[R0;R1;R2;R3]```).


### Execution:
_Inputs_ : 
```F#
(instr:Result<Parse<Instr>,string> option) dataPath machineMem memAddress 
```
_Output_ : 
```F#
Result<uint32*DataPath*Map<WAddr,uint32>,string> option
```

##### The ```execution``` function returns:

- The final address at which we stopped in memory.
- The map between registers and their stored values updated.
- The map between memory address and memory values updated.

Process :
- First checks the condition flags: if condition doesn't hold, returns the input registers, flags and memory.
- If conditions hold, the function matches successively the different types in the opcode: ```root/suffix```.
- It then calls computation functions (computeSTM or computeLDM) on the inputs, to which the "checkRestrictions" function has been applied: if restrictions have not been respected, it returns an error, otherwise, compute the instruction with those inputs.

- To do so, we use tail recursive functions (```modifyMapofRegs``` for ```LDM``` and ```modifyMapofStack``` for ```STM```):
	- Take the list of registers (example ```{R7, R0-R3}``` gave us ```[R0;R1;R2;R3;R7]```).
	- For ```STM```: Using ```Map.add```, update the input ``` Map<MemAddress,MemData>``` with the base Register Rn as Key for the memory address, and the data in the first register in the registers list (described above) as the data to store. 
As long as the registers list is not empty, call recursively ```modifyMapofStack``` on the tail of the registers list and with the new memory address being:  ```(previous memory address + or - 4)``` , depending on the suffix.

So _depending on the suffix_:

If suffix = ```IA```, call the function with ```(Initial Key for memory address) = Value in Rn``` ; call recursively by incrementing the memory address by 4.
If suffix = ```IB```, same as ```IA``` but ```(Initial Key for memory address) = (Value in Rn + 4)```.


If suffix = ```DA```, same as ```IA``` but instead of incrementing memory address, we decrement it by 4, AND we call it on the list of registers in reversed order, since when we decrement, we always want the top Register corresponding to the highest memory address.
If suffix = ```DB```, same as ```DA``` but ```(Initial Key for memory address) = (Value in Rn - 4)```.

The same reasoning applies almost exactly for ```LDM``` instructions, except that we modify the ```Map<RName,RegisterData>``` instead. 
Both ```LDM``` and ```STM``` functions also return ```PC``` incremented by 4, and if ```writeB``` is "true" then Rn is filled with the final memory address at which we stopped.

_Important things not to forget_:
- The base memory address must be superior to ```0x100``` (default value in Visual, can be modified) and divisible by 4 (for ```LDM/STM```).
- Restrictions are slightly different between ```LDM``` and ```STM``` instructions.
- ```LDM``` and ```STM``` only cost 4 cycles, whatever the number of registers that we need to store/load.
- Suffixes work differently between ```LDM``` and ```STM```, that is, ```STMDB``` and ```LDMDB``` will NOT work in the same way, and this because highest register is always at the highest memory location.
- The Map of memory location to data is chosen to be currently of type ``` Map<WAddr,uint32> ``` and not ``` Map<WAddr,MemLoc<'INS>> ``` to simplify testing, and because it is easy to modify in the group phase.



##### Contribution to the Group Deliverable : 

- Error messages were carefully chosen so that it is easy to find which part of the code threw an error.
- The output type of the ```execution``` function enables monadic error handling for the rest of the project.
- The ```execution``` function returns all the data (registers, memory, flags), and not just the one that has been altered: this enables it to be easily passed on to other function.
- _Clarity of code_ has sometimes been prioritised to _code efficiency_ (for example creating two different functions ```computeSTM``` and ```computeLDM``` to clearly see which function is doing which work, and how), enabling anyone starting to take over the code to quickly understand it.
- ```Root``` and ```Suffix``` types have been created separately to make the code more robust.




### Tests:

The ```test module``` is made mainly of unit tests. 

The ```testTokenizeandParse``` test checks that the ```tokenize``` and the ```makeRegList``` functions provide correct outputs for multiple inputs (repeated registers in registers list, lowercase letters, presence of ```!```).

The ```testAgainstVisual``` test compares ```Visual```'s and the ```execution``` function's outputs for an ```LDM``` instruction. I didn't manage to properly modify the Visual Test Files to correctly modify memory location and map them to the execution function's output for ```STM``` instructions. 
To do the ```LDM``` instruction: 
1) Type transformation functions are used to map ```Visual```'s output type to our ```Execution```'s output type: 
```makeMapofReg``` takes the ```(Out * int) list``` output of ```Visual``` and transforms it to a ```Map<RName;uint32>```.
```makeMapofMemLoc``` takes the ```vMemData``` output of Visual and a base memory address (that we set to ```0x1000``` since this is the value used in the default parameters for memory in ```Visual```); 
and creates a  ```Map<WAddr, <MemData>>```. 

Initial ```Map<RName;uint32>``` and ```Map<WAddr, <MemData>>``` are created by using the two above functions on the output of ```RunVisualWithFlagsOut defaultParas " " ``` .
_Important to note_: when comparing values; we don't look at R0, R13, R14 or R15 since these registers are used in ```Visual run through F#```, to store and load memory so they are not relevant. This explains the ```removeR0andR13toR15``` function.
_A lot of trouble arose when trying to set correctly memory locations, memory values because memory addresses seemed to be offset by some undetermined number_, which is why this test is trivial. But at least it can compare the two outputs.

2) Then, we call the same instructions on both our ```Execution``` function and ```RunVisualWithFlagsOut``` and compare their output registers.

The ```testRegMapAndMemMap``` is made of basic unit tests testing some ```LDM``` and ```STM```. Initial map of registers and map of memory location are created, and then instructions are executed using these maps as inputs. Compare the result with the expected output. To make it feasible, the length of the memory map is kept small.

Test Name			|   Command                 | Test |
--------------------------------| --------------------------|---------|
testAgainstVisual		|LDMIA R0, {R4-R6}	    |Passed
testTokenizeandParse		|tokenize "R8,{R0-R7,R6}"   |Passed
testTokenizeandParse		|tokenize "LR!,{R0-R3}"     |Passed
testTokenizeandParse		|makeRegList (tokenize "R7-LR,LR,LR,R3-R7}") |Passed
testTokenizeandParse		|makeRegList (tokenize "r10-sP,Pc}") |Passed
testRegMapAndMemMap		| STMIB R1, {R7,R8}         | Passed
testRegMapAndMemMap		| STMDB R12!, {R9-R11, R1}  | Passed
testRegMapAndMemMap		| LDMIA R1, {R2-R4}         | Passed
testRegMapAndMemMap		| LDMDA R3!, {R9,R1}        | Passed
























