# Civil-project-1
I wrote this code for one of my civil projects
"""
Structural Engineering Calculations according to British Standards
BS 8110, BS 5950, BS 6399
"""

import math
from dataclasses import dataclass
from typing import Tuple, List

@dataclass
class MaterialProperties:
    """Material properties according to British Standards"""
    concrete_grade: str = "C30"
    steel_grade: str = "500"
    modulus_elasticity_steel: float = 200000  # MPa
    modulus_elasticity_concrete: float = 28000  # MPa for C30 concrete

class BeamDesignBS8110:
    """
    Reinforced Concrete Beam Design according to BS 8110
    """
    
    def __init__(self, material_properties: MaterialProperties):
        self.material = material_properties
        
        # Partial safety factors BS 8110
        self.gamma_c = 1.5  # Concrete
        self.gamma_s = 1.15  # Steel
        
        # Characteristic strengths
        self.fcu = 30  # Concrete cube strength (MPa)
        self.fy = 500  # Steel yield strength (MPa)
    
    def calculate_moment_capacity_singly_reinforced(self, b: float, d: float, As: float) -> float:
        """
        Calculate moment capacity of singly reinforced rectangular beam
        
        Args:
            b: beam width (mm)
            d: effective depth (mm)
            As: area of tension reinforcement (mm²)
        
        Returns:
            Moment capacity (kN.m)
        """
        # Calculate steel stress
        fs = 0.87 * self.fy
        
        # Calculate depth of neutral axis
        x = (0.87 * self.fy * As) / (0.45 * self.fcu * b)
        
        # Check neutral axis depth limit (0.5d for singly reinforced)
        if x > 0.5 * d:
            raise ValueError("Section is over-reinforced. Consider doubly reinforced section.")
        
        # Calculate lever arm
        z = d - 0.45 * x
        
        # Calculate moment capacity
        Mu = 0.87 * self.fy * As * z / 1e6  # Convert to kN.m
        
        return Mu
    
    def calculate_required_reinforcement(self, M: float, b: float, d: float) -> float:
        """
        Calculate required tension reinforcement for given moment
        
        Args:
            M: design moment (kN.m)
            b: beam width (mm)
            d: effective depth (mm)
        
        Returns:
            Required area of reinforcement (mm²)
        """
        # Convert moment to N.mm
        M_design = M * 1e6
        
        # Calculate K factor
        K = M_design / (b * d**2 * self.fcu)
        
        # Check K limit
        K_max = 0.156  # For singly reinforced sections
        if K > K_max:
            raise ValueError("Section requires compression reinforcement")
        
        # Calculate lever arm
        z = d * (0.5 + math.sqrt(0.25 - K / 0.9))
        z = min(z, 0.95 * d)
        
        # Calculate required reinforcement
        As_required = M_design / (0.87 * self.fy * z)
        
        return As_required
    
    def calculate_shear_capacity(self, b: float, d: float, As: float) -> float:
        """
        Calculate shear capacity according to BS 8110
        
        Args:
            b: beam width (mm)
            d: effective depth (mm)
            As: area of tension reinforcement (mm²)
        
        Returns:
            Shear capacity (kN)
        """
        # Calculate design shear stress
        v_c = 0.79 * (100 * As / (b * d))**(1/3) * (400 / d)**(1/4) / 1.25
        
        # Shear capacity
        V_c = v_c * b * d / 1000  # Convert to kN
        
        return V_c

class SteelDesignBS5950:
    """
    Structural Steel Design according to BS 5950
    """
    
    def __init__(self, steel_grade: str = "S275"):
        self.py = 275  # Design strength (MPa) for S275 steel
        self.gamma_m0 = 1.0  # Partial safety factor
    
    def calculate_bending_capacity(self, section_modulus: float) -> float:
        """
        Calculate bending capacity of steel section
        
        Args:
            section_modulus: plastic section modulus (cm³)
        
        Returns:
            Moment capacity (kN.m)
        """
        # Convert cm³ to mm³
        Z = section_modulus * 1000
        
        # Moment capacity
        Mc = self.py * Z / 1e6  # kN.m
        
        return Mc
    
    def calculate_compression_capacity(self, area: float, compression_curve: str = "c") -> float:
        """
        Calculate compression capacity of steel member
        
        Args:
            area: cross-sectional area (mm²)
            compression_curve: buckling curve designation
        
        Returns:
            Compression capacity (kN)
        """
        # Simplified calculation (without slenderness effects)
        Pc = self.py * area / 1000  # kN
        
        return Pc

class LoadingBS6399:
    """
    Loading calculations according to BS 6399
    """
    
    @staticmethod
    def calculate_floor_loading(occupancy_type: str, imposed_load: float = None) -> float:
        """
        Calculate characteristic imposed load for floors
        
        Args:
            occupancy_type: type of occupancy
            imposed_load: specific load if provided (kN/m²)
        
        Returns:
            Characteristic imposed load (kN/m²)
        """
        load_values = {
            "residential": 1.5,
            "office": 2.5,
            "corridor": 3.0,
            "storage": 5.0,
            "parking": 2.5
        }
        
        if imposed_load:
            return imposed_load
        else:
            return load_values.get(occupancy_type.lower(), 2.0)
    
    @staticmethod
    def calculate_ultimate_load(dead_load: float, live_load: float) -> Tuple[float, float]:
        """
        Calculate ultimate design loads using partial safety factors
        
        Args:
            dead_load: characteristic dead load (kN/m²)
            live_load: characteristic live load (kN/m²)
        
        Returns:
            Tuple of (ultimate dead load, ultimate live load) in kN/m²
        """
        gamma_g = 1.4  # Dead load factor
        gamma_q = 1.6  # Live load factor
        
        ultimate_dead = dead_load * gamma_g
        ultimate_live = live_load * gamma_q
        
        return ultimate_dead, ultimate_live

def calculate_section_properties(b: float, h: float) -> dict:
    """
    Calculate section properties for rectangular section
    
    Args:
        b: width (mm)
        h: depth (mm)
    
    Returns:
        Dictionary of section properties
    """
    area = b * h  # mm²
    inertia = b * h**3 / 12  # mm⁴
    section_modulus = b * h**2 / 6  # mm³
    
    return {
        'area': area,
        'moment_of_inertia': inertia,
        'section_modulus': section_modulus,
        'radius_of_gyration': math.sqrt(inertia / area)
    }

# Example usage and testing
if __name__ == "__main__":
    # Initialize material properties
    materials = MaterialProperties(concrete_grade="C30", steel_grade="500")
    
    # Beam design example
    beam_design = BeamDesignBS8110(materials)
    
    # Calculate required reinforcement for a beam
    try:
        As_req = beam_design.calculate_required_reinforcement(
            M=150,  # kN.m
            b=300,  # mm
            d=450   # mm
        )
        print(f"Required reinforcement area: {As_req:.0f} mm²")
        
        # Calculate moment capacity
        Mu = beam_design.calculate_moment_capacity_singly_reinforced(
            b=300, d=450, As=942  # 3T20 bars
        )
        print(f"Moment capacity: {Mu:.1f} kN.m")
        
        # Calculate shear capacity
        Vc = beam_design.calculate_shear_capacity(b=300, d=450, As=942)
        print(f"Shear capacity: {Vc:.1f} kN")
        
    except ValueError as e:
        print(f"Design error: {e}")
    
    # Steel design example
    steel_design = SteelDesignBS5950("S275")
    Mc = steel_design.calculate_bending_capacity(section_modulus=500)  # 500 cm³
    print(f"Steel section moment capacity: {Mc:.1f} kN.m")
    
    # Loading example
    loading = LoadingBS6399()
    live_load = loading.calculate_floor_loading("office")
    ult_dead, ult_live = loading.calculate_ultimate_load(dead_load=2.0, live_load=live_load)
    print(f"Ultimate loads - Dead: {ult_dead} kN/m², Live: {ult_live} kN/m²")
    
    # Section properties
    props = calculate_section_properties(300, 600)
    print(f"Section modulus: {props['section_modulus']:.0f} mm³")
