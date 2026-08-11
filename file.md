{b.huggingface_dataset ? (
  
    className={styles.sourceLink}
    href={`https://huggingface.co/datasets/${b.huggingface_dataset}`}
    target="_blank" rel="noreferrer"
  >
    Source <ExternalLink size={12} />
  </a>
) : <span />}
