
# 1. Load the Model Checkpoint (e.g., Medium)
predictor = pretrained_mlip.get_predict_unit("uma-m-1p1", device="cuda")


Model Name,Description,Parameters (Active / Total),Use Case
uma-s-1p1,"Small (Default). The fastest model, suitable for most simulations while maintaining state-of-the-art accuracy.",~6.6M / 150M,"High-throughput screening, long MD trajectories."
uma-m-1p1,"Medium. Best-in-class accuracy across metrics, but slower and more memory-intensive than the small model.",~50M / 1.4B,High-precision single-point calculations or critical relaxations.
uma-l,Large,(Coming Soon),Not yet widely available for standard usage.

# 2. Attach the specific Task (e.g., Inorganic Materials)

Task Name,Domain / Application,Example Systems
oc20,Catalysis,"Adsorbates on surfaces (e.g., CO on Cu)."
omat,Inorganic Materials,"Bulk crystals, alloys, semiconductors."
omol,Molecules,"Small organic molecules, proteins, drugs."
odac,MOFs,"Metal-Organic Frameworks, carbon capture."
omc,Molecular Crystals,"Organic electronics, pharmaceutical crystals."

atoms.calc = FAIRChemCalculator(predictor, task_name="omat")
