sequenceDiagram
    participant BA as BackupAgent
    participant DD as DataDistributor  
    participant CP as CommitProxy
    participant BW as BackupWorker
    participant SS as StorageServer

    Note over BA,SS: Backup V3 Range Partitioned Logs - Initialization Flow

    BA->>BA: 1. Receive startBackup request
    BA->>DD: 2. Set key: backupPartitionRequired
    
    Note over DD: 3. Polling detects backupPartitionRequired
    DD->>DD: 4. Create KNOB_BACKUP_NUM_OF_PARTITIONS partitions
    
    DD->>CP: 5. Txn: new vector<Partition>
    DD->>CP: 6. Txn: remove backupPartitionRequired
    
    rect rgb(255, 248, 220)
        Note over CP: Version n begins
        CP->>CP: 7. Receive vector<Partition> @ Version=n
        CP->>CP: 8. Detect txnStateStore update needed
        CP->>CP: 9. Create PartitionMap: BackupTag ↔ vector<Partition>
        CP->>CP: 10. Update txnStateStore with PartitionMap @ Version=n
    end
    
    CP->>BW: 11. Tag PartitionMap to all BackupWorkers
    CP->>SS: 12. Tag PartitionMap to StorageServers (persist to disk)
    
    rect rgb(240, 248, 255)
        Note over CP,BW: Version n+1 begins
        CP->>CP: 13. Receive mutations @ Version=n+1
        CP->>CP: 14. Assign backup tags based on new PartitionMap
        CP->>BW: 15. Tag mutations to BackupWorkers
    end
    
    BW->>BW: 16. Peek TLog cursor
    BW->>BW: 17. Receive PartitionMap @ Version=n
    BW->>BW: 18. Store responsible partitions in-memory
    BW->>BW: 19. Create Version=n+1 directory structure
    
    rect rgb(245, 255, 245)
        Note over BW: Ongoing mutation processing
        BW->>BW: 20. Receive mutations @ Version=n+1
        BW->>BW: 21. Buffer mutations per partition
        BW->>BW: 22. After delay/file limit: upload partition buffers to S3
        BW->>BW: 23. Save progress to DB
    end

    Note over BA,SS: System now using range partitioned backup
