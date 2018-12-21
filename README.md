# repo-PDBUSQLExtractor
A repository for PDBUSQLExtractor for Azure Data Lake for parsing and extracting macromolecular data of protein structures.

Calculation of structural features of proteins, RNA molecules, and RNA-protein complexes on the basis of their geometries and studying various interactions within protein molecules, for which high-resolution structures are stored in Protein Data Bank (PDB), require parsing and extraction of suitable data stored in text files. To perform these operations on large scale in the face of the growing amount of macromolecular data in public repositories, we propose to perform them in the distributed environment of Azure Data Lake and scale the calculations on the Cloud. PDBUSQLExtractor is a set of dedicated data extractors for PDB files that can be used in various types of calculations performed over protein structures in the Azure Data Lake. Results of our tests show that the Cloud storage space occupied by the macromolecular data can be successfully reduced by using compression of PDB files without significant loss of data processing efficiency. Moreover, our experiments show that the performed calculations can be significantly accelerated when using large sequential files for storing macromolecular data and by parallelizing the calculations and data extractions that precede them. Finally, the PDBUSQLExtractor allows for all the protein- and RNA-related calculations that can be performed in a declarative way in U-SQL scripts for Data Lake Analytics.

In order to use the PDBUSQLExtractor a user must initially complete 3 steps:
1) set up the Azure Data Lake environment,
2) upload PDB files with protein structures into the Azure Data Lake Store, 
3) register the PDBUSQLExtractor library in the Azure Data Lake Analytics.

All of the mentioned operations can be completed through the Azure portal, Azure PowerShell, Azure CLI, or appropriate Application Programming Interfaces (APIs). Users should follow the Azure Data Lake Analytics Documentation for the particular method.

After completing the three necessary steps, the user can start developing the U-SQL scripts for data extraction and secondary data analysis. For the development of the U-SQL scripts, he can use any code editor, preferably Microsoft Visual Studio or Visual Studio Code.

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

```SQL
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

```SQL
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

```SQL
REFERENCE ASSEMBLY [PDBUSQLExtractor];

//-- S1.Data extraction
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

//-- S2.Selecting C-alpha atoms
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

//-- S3.Calculating the distance between C-alpha atoms and their coordinates, together with the index of the previous residue 
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


  
//-- S4.Storing results of S3 in csv file
OUTPUT @filteredDistances
TO "/output/distances.csv"
USING Outputters.Csv();
```
