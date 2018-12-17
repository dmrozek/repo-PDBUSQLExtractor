# repo-PDBUSQLExtractor
A repository for PDBUSQLExtractor for Azure Data Lake for parsing and extracting macromolecular data of protein structures

A part of a U-SQL script parsing and extracting data from the ATOM sections of the PDB files.


```SQL
REFERENCE ASSEMBLY [PDBUSQLExtractor];

@test =
        EXTRACT protId string,
                model int?,
                serial int?,
                name string,
                altLoc string,
                resNm string,
                chain string,
                resSq int?,
                iCode string,
                x double,
                y double,
                z double,
                occupancy double,
                tempFact double,
                element string,
                charge string
        FROM "/input/decompressed/{*.ent}"
        USING new PDBUSQLExtractor.PDBExtractor("ATOM");

OUTPUT @test
TO "/output/resultatom.tsv"
USING Outputters.Tsv();
```

Sample U-SQL script parsing and extracting data from the SEQRES  sections of PDB files.

```
REFERENCE ASSEMBLY [PDBUSQLExtractor];

@test =
    EXTRACT proteinId string,
            serNum int?,
            chainId string,
            numRes int?,
            resName1 string,
            resName2 string,
            resName3 string,
            resName4 string,
            resName5 string,
            resName6 string,
            resName7 string,
            resName8 string,
            resName9 string,
            resName10 string,
            resName11 string,
            resName12 string,
            resName13 string
        FROM "/input/decompressed/{*.ent}"
        USING new PDBUSQLExtractor.PDBExtractor("SEQRES");

OUTPUT @test
TO "/output/resultatom.tsv"
USING Outputters.Tsv();
```

Extraction of the ATOM section data from PDB data sets assembled in sequential files with the use of the PDBConcatExtractor extractor.

```
REFERENCE ASSEMBLY [PDBUSQLExtractor];

@test =
        EXTRACT proteinId string,
        ...
        FROM "/input/pdbdataset.ent.gz"
        USING new PDBUSQLExtractor.PDBConcatExtractor("ATOM");
OUTPUT @test
TO "/output/resultatom.txt"
USING Outputters.Tsv();
```


Sample U-SQL script that extracts data from PDB files and produces the output rowset containing distances between C-alpha atoms within protein structures: 

```
REFERENCE ASSEMBLY [PDBUSQLExtractor];

// S1.Data extraction
@extracted =
    EXTRACT proteinId string,
            modelId int,
            serial int,
            name string,
            altLoc string,
            resName string,
            chainID string,
            resSeq int,
            iCode string,
            x double,
            y double,
            z double,
            occupancy double,
            tempFactor double,
            element string,
            charge string
    FROM "/input/pdbdataset.ent.gz"
    USING new PDBUSQLExtractor.PDBConcatExtractor("ATOM");

// S2.Selecting C-alpha atoms
@filteredAtoms =
    SELECT proteinId,
           resSeq,
           x,
           y,
           z,
           modelId,
           iCode,
           chainID,
           resName
    FROM @extracted
    WHERE name.Trim() == "CA";

// S3.Calculating the distance between C-alpha atoms and their coordinates, together with the index of the previous residue 
@filteredDistances =
    SELECT a1.proteinId, a1.chainID, a1.resName,
           Math.Sqrt(Math.Pow(a2.x - a1.x, 2) + Math.Pow(a2.y - a1.y, 2) 
             + Math.Pow(a2.z - a1.z, 2)) AS CaCaDistance,
           a1.resSeq AS resSeq1,
           a2.resSeq AS resSeq2,
           a1.modelId
    FROM @filteredAtoms AS a1 JOIN @filteredAtoms AS a2
         ON a1.proteinId == a2.proteinId
    WHERE a1.resSeq == a2.resSeq + 1 AND a1.modelId == a2.modelId 
         AND a1.chainID == a2.chainID;


  
// S4.Storing results of S3 in csv file
OUTPUT @filteredDistances
TO "/output/distances.csv"
USING Outputters.Csv();
```
